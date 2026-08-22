# Parte 3: Escalado, Maduración y Evolución de su Plataforma

## Capítulo 11: Validación del Cumplimiento Normativo y Políticas como Código

Una vez implementados los *starter kits* para proporcionar patrones arquitectónicos probados a los usuarios, es necesario abordar los desafíos de gobernanza y seguridad asociados. Un escenario habitual ocurre cuando un desarrollador despliega un contenedor sin límites de recursos, extrae imágenes desde registros no confiables o introduce secretos codificados directamente (*hardcoded*).

En este capítulo abordaremos el **cumplimiento normativo (*compliance*) como una capacidad central de la plataforma**. Si la seguridad y el cumplimiento se tratan como una ocurrencia tardía, los equipos experimentarán fricciones constantes. Implementaremos políticas como código (*Policy-as-Code*) utilizando **Open Policy Agent (OPA)** y su variante integrada en Kubernetes, **Gatekeeper**.

Al finalizar este capítulo, serás capaz de:
- Desplegar y configurar OPA Gatekeeper como controlador de admisión en tu clúster de Kubernetes.
- Crear políticas de validación utilizando el lenguaje declarativo **Rego**.
- Implementar pruebas de desplazamiento a la izquierda (*shift-left testing*) con **Conftest** para detectar infracciones antes del despliegue.
- Diseñar y publicar paneles de control de cumplimiento en **Grafana** para visibilidad de los interesados.
- Equilibrar la aplicación estricta de políticas con la experiencia de desarrollo (DevEx) para minimizar la fricción.

---

### Despliegue de un Controlador de Admisión (*Admission Controller*)

En Kubernetes, la gobernanza se aplica mediante controladores de admisión (*admission controllers*) [1], componentes que interceptan las solicitudes a la API antes de persistir los objetos en etcd. Existen dos tipos de webhooks de admisión:
1. **Webhooks de Validación (*Validating Webhooks*)**: Evalúan la solicitud y deciden si se admite o se rechaza.
2. **Webhooks de Mutación (*Mutating Webhooks*)**: Modifican los objetos antes de su creación para garantizar estándares por defecto.

**OPA Gatekeeper** permite definir políticas desacopladas del código interno de la API mediante dos primitivas de Kubernetes:
- **`ConstraintTemplate`**: Define la lógica de la política en Rego y el esquema de parámetros aceptados.
- **`Constraint`**: Instancia un `ConstraintTemplate`, especificando los recursos a los que aplica, namespaces excluidos y parámetros de configuración.

> **Figura 11.1** - Arquitectura del controlador de admisión OPA Gatekeeper

#### 1. Definición del ConstraintTemplate (`constraint-template.yaml`)

```yaml
apiVersion: constraints.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
      validation:
        openAPIV3Schema:
          properties:
            message:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests
          msg := sprintf("Container %v is missing resource requests", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits
          msg := sprintf("Container %v is missing resource limits", [container.name])
        }
```

#### 2. Definición del Constraint (`constraint.yaml`)

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resources
spec:
  match:
    excludedNamespaces:
      - kube-system
      - kube-public
      - gatekeeper-system
  parameters:
    message: "All containers must have resource requests and limits"
```

Aplicación en el clúster:

```bash
kubectl apply -f constraint-template.yaml
kubectl apply -f constraint.yaml
```

Si se intenta desplegar un pod sin límites de recursos (`test-pod.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

El webhook de validación intercepta la solicitud y la rechaza inmediatamente en el límite de la API:

```bash
$ kubectl apply -f test-pod.yaml
error: admission webhook "validation.gatekeeper.sh" denied the request: [require-resources] container <nginx> is missing resource limits
```

---

### Políticas como Código: ¿Habilitar o Forzar? (*Enabler vs Enforcer*)

El enfoque recomendado para la adopción de políticas se basa en una progresión deliberada:
1. **Fase de Auditoría y Educación**: Desplegar políticas en modo auditoría (`enforcementAction: dryrun`) y publicar métricas de incumplimiento en paneles de control.
2. **Aplicación Progresiva**: Pasar a modo restrictivo (`deny`) solo aquellas políticas que representen un riesgo de seguridad crítico (ej. evitar escalada de privilegios o ejecución como root).
3. **Escucha y Retroalimentación**: Si los desarrolladores solicitan excepciones masivas a una política, suele indicar que está mal diseñada o insuficientemente explicada.

---

### Uso de Rego para la Escritura de Políticas

**Rego** [3] es un lenguaje de consultas declarativo diseñado para expresar reglas sobre estructuras de datos jerárquicas (JSON/YAML).

#### 1. Política para Prohibir Contenedores Ejecutándose como Root

```rego
package k8snoroot

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  container.securityContext.runAsUser == 0
  msg := sprintf("Container %v is running as root", [container.name])
}
```

- `package k8snoroot`: Espacio de nombres de la política.
- `violation[{"msg": msg}]`: Define el conjunto de violaciones detectadas.
- `container := input.review.object.spec.containers[_]`: Itera sobre cada contenedor dentro del pod.
- `container.securityContext.runAsUser == 0`: Evalúa si el identificador de usuario es `0` (root).

#### 2. Restricción de Registros de Imágenes Permitidos

```rego
allowed_registries := ["gcr.io", "docker.io/library", "mycompany.azurecr.io"]

violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  image := container.image
  not any_allowed(image)
  msg := sprintf("Image %v is from an unapproved registry", [image])
}

any_allowed(image) {
  allowed := allowed_registries[_]
  startswith(image, allowed)
}
```

#### 3. Plantilla de Seguridad Integral (`k8ssecuritybaselines`)

```yaml
apiVersion: constraints.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8ssecuritybaselines
spec:
  crd:
    spec:
      names:
        kind: K8sSecurityBaselines
      validation:
        openAPIV3Schema:
          properties:
            message:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8ssecuritybaselines

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Container %v is running in privileged mode", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.readOnlyRootFilesystem
          msg := sprintf("Container %v should have a read-only root filesystem", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.capabilities.add[_] == "SYS_ADMIN"
          msg := sprintf("Container %v is requesting dangerous Linux capabilities", [container.name])
        }
```

---

### Pruebas de Desplazamiento a la Izquierda (*Shift-Left Policy Testing*)

Los desarrolladores no deben esperar a enviar código al clúster para descubrir si sus manifiestos son conformes.

> **Figura 11.2** - Capas de aplicación de políticas desde el desarrollo local hasta producción

**Conftest** [2] permite evaluar manifiestos de Kubernetes de forma local, en hooks de *pre-commit* o en canalizaciones de CI/CD utilizando las mismas políticas Rego de OPA.

#### 1. Definición de Política para Conftest (`policy/resources.rego`)

```rego
# policy/resources.rego
package main

deny contains msg if {
  container := input.spec.containers[_]
  not container.resources.requests
  msg := sprintf("Container %v missing resource requests", [container.name])
}

deny contains msg if {
  container := input.spec.containers[_]
  not container.resources.limits
  msg := sprintf("Container %v missing resource limits", [container.name])
}
```

#### 2. Validación Local contra Manifiestos (`deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
          image: myregistry/web-app:v1
```

Ejecución de la prueba:

```bash
conftest test deployment.yaml -p policy/
```

Resultado:

```text
FAIL - deployment.yaml - Container web missing resource requests
FAIL - deployment.yaml - Container web missing resource limits

2 tests, 0 passed, 2 failed
```

#### 3. Integración en Hooks de Pre-Commit (`.pre-commit-config.yaml`)

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/open-policy-agent/conftest
    rev: v0.67.0
    hooks:
      - id: conftest
        args:
          - test
          - -p
          - policy/
          - --file-type
          - yaml
```

#### 4. Integración en GitHub Actions (`.github/workflows/validate.yml`)

```yaml
# .github/workflows/validate.yml
name: Policy Validation
on: [pull_request]

jobs:
  conftest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install conftest
        run: |
          wget -q https://github.com/open-policy-agent/conftest/releases/download/v0.67.0/conftest_0.67.0_Linux_x86_64.tar.gz
          tar xzf conftest_0.67.0_Linux_x86_64.tar.gz
          sudo mv conftest /usr/local/bin/
      - name: Run policy tests
        run: conftest test manifests/ -p policy/
```

Para retroalimentación visual directa en los comentarios del Pull Request, utiliza el formato nativo de GitHub:

```bash
conftest test manifests/ -p policy/ --output github
```

---

### Herramientas de Gobernanza Cloud frente a OPA Gatekeeper

- **Herramientas Nativas de la Nube (AWS Security Hub, Azure Policy, Google Policy Controller)**: Excelentes para gobernar recursos propios del proveedor de nube (IAM, VPCs, almacenamiento).
- **OPA Gatekeeper**: Es agnóstico a la nube, portátil y unifica las políticas de Kubernetes tanto en nubes públicas (EKS, AKS, GKE) como en clústeres locales (*on-premises*).

---

### Paneles de Control de Cumplimiento (*Compliance Dashboards*)

Gatekeeper audita continuamente los recursos existentes en el clúster. La información de auditoría se expone en eventos de Kubernetes y métricas de Prometheus.

#### 1. Configuración de Prometheus para Raspar Métricas de Gatekeeper (`gatekeeper.yml`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-gatekeeper
  namespace: monitoring
data:
  gatekeeper.yml: |
    global:
      scrape_interval: 30s
    scrape_configs:
      - job_name: 'gatekeeper'
        static_configs:
          - targets: ['gatekeeper-audit.gatekeeper-system:8888']
```

#### 2. Exportador de Eventos de Infracción a Prometheus (`gatekeeper_exporter.py`)

```python
#!/usr/bin/env python3
import os
from kubernetes import client, config, watch
from prometheus_client import start_http_server, Counter, Gauge

config.load_incluster_config()
v1 = client.CoreV1Api()

violation_counter = Counter(
    'gatekeeper_violations_total',
    'Total policy violations',
    ['constraint', 'namespace']
)

def watch_violations():
    w = watch.Watch()
    for event in w.stream(
        v1.list_event_for_all_namespaces,
        field_selector='reason=ConstraintViolation'
    ):
        event_obj = event['object']
        constraint = event_obj.involved_object.name
        namespace = event_obj.involved_object.namespace or 'cluster-level'
        violation_counter.labels(
            constraint=constraint,
            namespace=namespace
        ).inc()

if __name__ == '__main__':
    start_http_server(8000)
    watch_violations()
```

#### 3. Definición de Paneles en Grafana [4]

```json
{
  "dashboard": {
    "title": "Gatekeeper Compliance",
    "panels": [
      {
        "title": "Total Violations",
        "targets": [
          {
            "expr": "increase(gatekeeper_violations_total[5m])"
          }
        ]
      },
      {
        "title": "Violations by Constraint",
        "targets": [
          {
            "expr": "topk(5, sum by (constraint) (gatekeeper_violations_total))"
          }
        ]
      },
      {
        "title": "Violations by Namespace",
        "targets": [
          {
            "expr": "topk(10, sum by (namespace) (gatekeeper_violations_total))"
          }
        ]
      },
      {
        "title": "Compliance Rate",
        "targets": [
          {
            "expr": "(count(gatekeeper_constraint_status{status='enforced'}) / count(gatekeeper_constraint_status)) * 100"
          }
        ]
      }
    ]
  }
}
```

> **Integración con Backstage**: Las métricas `gatekeeper_violations_total` y `gatekeeper_audit_last_run_time` pueden mostrarse en tarjetas de entidad dentro de Backstage, ofreciendo a los desarrolladores una tarjeta de puntuación (*scorecard*) en vivo de cumplimiento en su propio portal.

---

### Ejercicio 11.1: Aplicación de Políticas sobre la Aplicación de Demostración

1. Crear `ConstraintTemplates` para recursos, contextos de seguridad y registros autorizados.
2. Validar los manifiestos de la aplicación de demostración mediante `conftest`.
3. Desplegar Gatekeeper y aplicar las restricciones (`Constraints`) en el clúster.
4. Ajustar los manifiestos de la aplicación hasta lograr total conformidad.
5. Generar un informe de cumplimiento mediante el script exportador y paneles de Grafana.

---

### Resumen

- **Políticas como Código Proactivas**: Transforma el cumplimiento de auditorías reactivas a una validación automatizada en tiempo real.
- **Control de Admisión Declarativo**: OPA Gatekeeper evalúa las solicitudes antes de su creación mediante `ConstraintTemplates` y `Constraints`.
- **Desplazamiento a la Izquierda con Conftest**: Permite a los desarrolladores detectar y corregir infracciones en sus máquinas locales y en CI/CD antes del despliegue.
- **Visibilidad Centralizada**: La exportación de métricas a Prometheus y Grafana permite monitorizar la tasa de cumplimiento y la remediación continua.
- **Adopción Progresiva**: Comenzar en modo auditoría para educar antes de aplicar bloqueos estrictos que puedan interrumpir la entrega.

---

### Referencias

- **[1]** *Kubernetes Admission Controllers Reference*. [https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- **[2]** *Conftest — Testing Configuration Files with Open Policy Agent*. [https://www.conftest.dev/](https://www.conftest.dev/)
- **[3]** *The Rego Policy Language Documentation*. [https://www.openpolicyagent.org/docs/policy-language](https://www.openpolicyagent.org/docs/policy-language)
- **[4]** *Grafana Dashboards and Visualizations*. [https://grafana.com/](https://grafana.com/)
