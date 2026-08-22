# Parte 1: Diseño, Construcción y Despliegue de la Plataforma de Ingeniería Principal

## Capítulo 3: Asegurando el Acceso a la Plataforma

Con un entorno de ejecución de plataforma en marcha que cuenta con una malla de servicios configurada para que la utilicen los desarrolladores de NewTech, María se enfrenta a la presión de su responsable técnico de producto (*Technical Product Owner*, TPO) para que los equipos comiencen a desplegar en ella. Solo unos pocos equipos solicitan desplegar debido a problemas pasados al gestionar su propia infraestructura. Sin embargo, los líderes empresariales buscan obtener valor para la organización lo más rápido posible a partir de la inversión realizada en el equipo de plataforma que lidera María. 

Por su parte, el equipo de plataforma siente que apenas ha comenzado a construirla, por lo que teme que no esté lista para un uso real y le pide a María que frene la presión. Para encontrar un punto intermedio, María sugiere que el TPO implemente un sistema de permisos que proteja los componentes del sistema de la plataforma frente a daños accidentales por parte de los equipos de desarrollo. Una vez establecido esto, también deberían disponer de una aplicación de demostración que puedan desplegar en la plataforma utilizando los mismos permisos que se otorgarán a los equipos de desarrollo. De ese modo, su equipo podrá comprender de primera mano los desafíos a los que se enfrentarán los desarrolladores y qué tipo de soporte deberán proporcionar.

El TPO acepta la propuesta y le informa de que ya se ha identificado un equipo piloto. El objetivo es incorporarlos a la plataforma tan pronto como estas capacidades estén listas, de modo que se puedan recopilar comentarios tempranos y ayudar a priorizar el backlog. La recomendación es crear una aplicación de demostración que reproduzca la arquitectura de lo que desplegará el equipo piloto, ya que será un caso de uso habitual cuando más equipos comiencen a utilizar la plataforma: una aplicación web de una sola página (*Single-Page Application*, SPA) servida en Internet mediante TLS con un backend de microservicios. 

Idealmente, la emisión y gestión de los certificados TLS debería gestionarse en nombre de los equipos para cumplir con las políticas de la empresa, pero los equipos deben tener la libertad de definir cómo desean enrutar hacia su página desde la dirección de dominio de la compañía. Una vez identificados los requisitos, ¡es hora de preparar la plataforma para usuarios reales!

Al final de este capítulo, utilizaremos esta información para:

- Habilitar el inicio de sesión en el clúster tanto para usuarios como para cuentas de servicio utilizando controles de acceso basados en roles (RBAC).
- Comprender la experiencia de usuario de la plataforma actual mediante el despliegue de una aplicación de demostración.

---

### Comprensión de los Requisitos de Seguridad de la Plataforma

Como se describió en la introducción, el equipo de María necesita equilibrar la seguridad con la velocidad de desarrollo. El primer paso es comprender qué elementos requieren protección y seguridad.

Una auditoría rápida del inventario de seguridad del clúster se puede realizar mediante un script sencillo como este:

```bash
#!/bin/bash
# Quick security audit for Maria's team
echo "Cluster Security Inventory"

# Find overly permissive RoleBindings
kubectl get clusterrolebindings -o json | jq -r '
  .items[] | select(.roleRef.name=="cluster-admin") |
  .metadata.name + ": " + (.subjects[].name // "unknown")'

# Identify service accounts with secrets
kubectl get serviceaccounts -A -o json | jq -r '
  .items[] | select(.secrets|length > 0) |
  .metadata.namespace + "/" + .metadata.name'

# Check for pods running as root
kubectl get pods -A -o json | jq -r '
  .items[] | select(.spec.securityContext.runAsUser == 0) |
  .metadata.namespace + "/" + .metadata.name + " (ROOT!)"'
```

#### El Dilema entre Seguridad y Velocidad (*Security–Velocity Tradeoff*)

Una vez que las capacidades de la plataforma están en manos de los desarrolladores, los ingenieros de plataforma, por diseño, tienen una interacción diaria mínima con los desarrolladores reales que utilizan sus soluciones, más allá del registro de API u otros mecanismos de seguimiento avanzados. Incluso entonces, la percepción de su trabajo por parte de los usuarios finales no siempre resulta evidente para los creadores. Si los desarrolladores experimentan cambios incompatibles o interrupciones que obstaculizan su trabajo de desarrollo activo, a menudo temen que la propia plataforma sea la causa. 

La documentación es otro aspecto que genera sobrecarga en el equipo de plataforma, ya que puede convertirse en otra barrera ante un uso incorrecto. Todo esto puede llevar a situaciones en las que los propios equipos de plataforma se resistan al empuje de los gerentes de producto para lanzar su MVP (*Minimum Viable Platform*). Sin embargo, en cuanto se aprueba la financiación para el equipo de plataforma, surge una presión constante para medir su valor (el ROI) y demostrar que las inversiones han merecido la pena. Aunque la seguridad no siempre haya sido el factor principal que impulsó la inversión en plataformas, en escenarios específicos desempeñará un papel decisivo en el retorno de la inversión.

Debido a este conflicto inherente, la mayoría de las organizaciones recurren a un término medio: seleccionar un **equipo piloto** razonablemente informado, receptivo y colaborador que mire más allá de las asperezas iniciales y esté dispuesto a proporcionar retroalimentación detallada. Esto brinda a los equipos de plataforma la flexibilidad de continuar construyendo con los mismos acuerdos e interfaces que enfrentan los equipos piloto, permitiendo identificar puntos de fricción antes de que bloqueen a los usuarios reales.

En lugar de limitarse a describir qué sucedería, resulta mucho más revelador examinar lo que se debe evitar en cuanto a atajos de seguridad. Veamos el siguiente ejemplo de un atajo peligroso:

```yaml
# The cluster-admin shortcut Maria's team must avoid
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-cd-pipeline
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ci-cd-does-everything # Red flag!
subjects:
  - kind: ServiceAccount
    name: ci-cd-pipeline
    namespace: default
roleRef:
  kind: ClusterRole
  name: cluster-admin # "Convenient" but dangerous
  apiGroup: rbac.authorization.k8s.io
```

#### Modelado de Amenazas para Plataformas Internas

A nivel interno en una organización, ocurren muchos más incidentes de seguridad relacionados con errores de configuración o confusión que con intentos maliciosos deliberados. Que los desarrolladores intenten desplegar en un espacio de nombres (*namespace*) restringido es un error común. Aunque puede no ser tan dañino como que usuarios con permisos excesivos eliminen accidentalmente infraestructura crítica (como un controlador Ingress o el DNS del clúster), el impacto en la percepción y el miedo a cometer fallos aumenta si no existen las barreras de seguridad adecuadas. Gran parte de esto se debe a pruebas y observabilidad inadecuadas provistas por plataformas que dejan margen al error del desarrollador. 

Un fallo habitual que vemos con frecuencia es que la canalización de CI/CD utilice una única cuenta de servicio muy permisiva con permisos de `cluster-admin`, simplemente porque se consideró "cómodo". En ocasiones, estos tokens de cuenta de servicio sin expiración se filtran a Git o se imprimen en los registros durante tareas de depuración, creando una situación de compromiso de seguridad.

Otros retos habituales incluyen:
- Aplicaciones en un namespace accediendo a secretos o servicios de otro.
- Políticas de red sin configurar.
- Compartición indebida de volúmenes de almacenamiento.
- Cuotas de recursos sin aplicar.
- Exposición de tráfico sin cifrar.

#### El Principio de Mínimo Privilegio en la Práctica

¿Cómo gestionamos estas situaciones cuando los equipos de plataforma están bajo una gran presión? La solución estándar es el **principio de mínimo privilegio**. Según el rol de cada persona, los permisos deben ajustarse estrictamente a sus responsabilidades:

- Un perfil de **`platform-admin`** necesita visibilidad completa del clúster para depurar problemas de plataforma, pero no requiere acceso a los secretos de aplicación de los equipos.
- Un perfil de **`platform-user`** debe poder desplegar y depurar sus propias aplicaciones, pero no ver las cargas de trabajo de otros equipos ni modificar los componentes de la plataforma.
- Se deben incorporar mecanismos de excepción controlados (*escape hatches*) mediante escalada de privilegios temporal con registro, auditoría y reversión automatizados.

> **Figura 3.1** - Perfiles de acceso a la plataforma y relaciones

Las cuentas de servicio automatizadas (como las de CI/CD) solo necesitan permisos de despliegue en un namespace específico, nunca acceso administrativo a nivel de clúster. Del mismo modo, los sistemas de monitorización solo requieren acceso de lectura a métricas.

Los patrones de acceso limitados en el tiempo (*time-bound access*) son otro estándar: los tokens OAuth suelen expirar tras periodos cortos (por ejemplo, 15 minutos), forzando reautenticaciones periódicas. Cada acción en el clúster debe registrarse con su identidad correspondiente, y los cambios en políticas RBAC deben generar alertas de seguridad con registros inmutables retenidos según marcos como **SOC 2**, **HIPAA** o **PCI-DSS** [2].

---

### Gestión de Identidades y Accesos con OAuth

¿Por qué utilizar OAuth para Kubernetes en lugar de certificados X.509 tradicionales? Los certificados X.509 presentan limitaciones fundamentales: carecen de un flujo de revocación integrado (revocar un certificado comprometido exige reconfigurar el clúster), no admiten MFA de forma nativa ni políticas de acceso condicional, y la gestión individual de certificados se vuelve inviable a medida que los equipos crecen.

Al federar identidades con OAuth/OIDC [1] (conectando con Active Directory, Okta, Azure AD, Google Workspace, etc.), los usuarios se autentican con sus credenciales corporativas existentes, heredan políticas organizacionales y disfrutan de inicio de sesión único (SSO). Además, los tokens de acceso de corta duración reducen la ventana de exposición en caso de compromiso.

Para detectar tokens expuestos en registros de CI/CD, se pueden utilizar scripts de validación como el siguiente:

```bash
#!/bin/bash
# scan-for-tokens.sh - Run in CI pipeline

# Check for leaked service account tokens
grep -r "eyJhbGciOi" build.log && {
  # Base64-encoded beginning of a JWT
  echo "ALERT: Possible token leak detected!"
  exit 1
}

# Check for kubeconfig artifacts
find . -name "*.kubeconfig" -o -name "*config*.yaml" | while read f; do
  grep -q "client-certificate-data" "$f" && {
    echo " Found kubeconfig: $f"
    exit 1
  }
done
```

#### Instalación y Configuración de Keycloak

**Keycloak** [3] es una solución de código abierto de gestión de identidades y accesos que proporciona autenticación y autorización centralizadas con SSO. Suele desplegarse como un `StatefulSet` en el namespace `platform-services` con almacenamiento persistente y una base de datos PostgreSQL de respaldo (utiliza H2 por defecto).

El proceso de configuración incluye:
1. Crear un reino (*realm*) dedicado llamado `"kubernetes"` para aislar la autenticación del clúster.
2. Configurar un cliente OAuth para el servidor de la API de Kubernetes.
3. Definir grupos de usuarios (`platform-admins` y `platform-users`) que se mapearán a los roles de Kubernetes.
4. Establecer el tiempo de expiración del token de acceso en 15 minutos.
5. Configurar el servidor de la API de Kubernetes con los parámetros `--oidc-issuer-url`, `--oidc-client-id`, `--oidc-username-claim` y `--oidc-groups-claim` para mapear los grupos de Keycloak a grupos RBAC de Kubernetes.

#### Configuración de OIDC y Creación de Identidades de Plataforma

Al conectar Keycloak con el servidor de la API de Kubernetes, el servidor descarga las claves criptográficas públicas de Keycloak y las utiliza para verificar que cualquier token presentado haya sido emitido de forma legítima y no haya sido manipulado.

Cuando un usuario inicia sesión a través de Keycloak, su token JWT incluye automáticamente sus pertenencias a grupos. Al realizar una petición con `kubectl`, el servidor de la API lee los grupos del token y aplica los permisos correspondientes. Añadir un nuevo desarrollador a la plataforma se reduce a incluirlo en el grupo `platform-users` en Keycloak, otorgándole acceso inmediato sin necesidad de reconfigurar el clúster.

---

### Implementación del Control de Acceso Basado en Roles (RBAC)

En Kubernetes, RBAC define qué acciones están permitidas sobre qué recursos:
- Un **`Role`** y un **`ClusterRole`** definen permisos a nivel de namespace o a nivel de clúster completo, respectivamente.
- Un **`RoleBinding`** y un **`ClusterRoleBinding`** vinculan dichos permisos a usuarios, grupos o cuentas de servicio.
- Si un permiso no se concede explícitamente, se deniega por defecto (arquitectura de Confianza Cero).

| Recurso / Acción | `platform-admin` | `platform-user` | `Cuenta de Servicio CI/CD` |
| :--- | :--- | :--- | :--- |
| **Namespaces** | | | |
| Listar todos los namespaces | Sí | Solo el namespace propio | No |
| Crear / Eliminar namespaces | Sí | No | No |
| **Cargas de Trabajo (*Workloads*)** | | | |
| Ver en namespace propio | Sí | Sí | Solo lectura |
| Crear / Actualizar cargas de trabajo (*create, update, patch*) | Sí | Sí | Solo despliegues (*Deployments*) |
| Ver namespaces del sistema (*kube-system*, *istio-system*) | Sí | No | No |

> **Tabla 3.1** - Matriz de permisos RBAC

#### Protección del Acceso por CLI y Flujos de Inicio de Sesión

Separar la autenticación de la autorización permite inicios de sesión seguros desde servidores desatendidos (*headless*), contenedores o entornos restringidos utilizando proveedores de identidad corporativos. Al ejecutar `kubectl login`, la gestión del código de dispositivo, el intercambio de tokens y la generación de `kubeconfig` ocurren de forma transparente.

#### Protección de CI/CD con Cuentas de Servicio

Las canalizaciones de CI/CD son objetivos de ataque críticos (como demostró el incidente de CircleCI en 2023 [4]). Para protegerlas:
- Cada aplicación debe contar con su propia cuenta de servicio limitada a un único namespace y con permisos restringidos exclusivamente a sus funciones de despliegue.
- Se deben utilizar tokens proyectados de corta duración con expiración automática en lugar de cuentas de servicio permanentes.
- Las cuentas de servicio pueden hacer referencia a secretos en los manifiestos de despliegue sin necesidad de tener permisos para leer los secretos directamente, evitando que se expongan en los registros de compilación.

#### Barreras de Seguridad con Políticas como Código (OPA Gatekeeper)

Mientras que RBAC controla *quién* puede realizar acciones, **Open Policy Agent (OPA) Gatekeeper** [5] evalúa *cómo* están configurados los recursos en el momento de la admisión (*admission time*), impidiendo la entrada de malas configuraciones.

Despliega OPA Gatekeeper:

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.14.0/deploy/gatekeeper.yaml
```

Plantilla de restricción para exigir límites de recursos de CPU y memoria (`template-resource-limits.yaml`):

```yaml
# template-resource-limits.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: resourcelimits
spec:
  crd:
    spec:
      names:
        kind: ResourceLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            cpu:
              type: string
              description: "Maximum CPU limit allowed per container (e.g.'2')"
            memory:
              type: string
              description: "Maximum Memory limit allowed per container (e.g.'4Gi')"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package resourcelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' missing memory limits", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' missing CPU limits", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          mem_limit := container.resources.limits.memory
          exceeds_max_memory(mem_limit)
          msg := sprintf("Container '%v' memory limit %v exceeds maximum", [container.name, mem_limit])
        }

        exceeds_max_memory(mem) {
          mem_value := units.parse_bytes(mem)
          max_value := units.parse_bytes(input.parameters.memory)
          mem_value > max_value
        }
```

Restricción para requerir etiquetas de gobernanza en los namespaces (`constraint-namespace-labels.yaml`):

```yaml
# constraint-namespace-labels.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: K8sRequiredLabels
metadata:
  name: namespace-must-have-team
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
    excludedNamespaces:
      - "kube-system"
      - "kube-public"
      - "kube-node-lease"
      - "gatekeeper-system"
      - "flux-system"
      - "istio-system"
      - "platform-services"
      - "cert-manager"
      - "monitoring"
  parameters:
    labels:
      - key: "team"
        allowedRegex: "^[a-z]{2,20}$"
      - key: "environment"
        allowedRegex: "^(dev|staging|prod)$"
      - key: "cost-center"
        allowedRegex: "^[0-9]{4,6}$"
```

Prueba de violación de políticas y alerta en Prometheus:

```yaml
# Test: Deploy without resource limits (will be rejected)
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-deployment
  namespace: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bad
  template:
    metadata:
      labels:
        app: bad
    spec:
      containers:
        - name: nginx
          image: nginx:latest # Also violates registry policy
          # Missing resource limits!
EOF
# Expected output:
# Error from server ([denied by resourcelimits] Container 'nginx' missing memory limits
# [denied by trustedregistries] Container image 'nginx:latest' not from approved registries)
```

```yaml
# prometheus-rule-violations.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: opa-violations
  namespace: gatekeeper-system
spec:
  groups:
    - name: opa.rules
      interval: 30s
      rules:
        - alert: HighPolicyViolationRate
          expr: rate(gatekeeper_violation_total[5m]) > 0.1
          annotations:
            summary: "High rate of policy violations ({{ $value }} per second)"
            description: "Teams are frequently attempting forbidden operations"
```

Estas políticas impiden que los usuarios:
- Desplieguen contenedores desde registros no confiables.
- Ejecuten contenedores privilegiados o como usuario `root`.
- Desplieguen cargas de trabajo sin límites de recursos.
- Creen namespaces sin etiquetas para seguimiento de costes.

---

### Construcción de la Experiencia de la Aplicación de Demostración

Crear una aplicación de demostración con las mismas restricciones que enfrentará el equipo piloto revela fricciones antes de que afecten a los desarrolladores reales.

> **Figura 3.2** - Arquitectura de la aplicación de demostración (demo-app)

El tráfico externo se enruta a través de un **Istio Gateway** y un **VirtualService** hacia los componentes de la aplicación. TLS es administrado automáticamente por un recurso `Certificate` de **cert-manager** en el namespace `istio-system`. 

> **Importante**: El namespace `demo-app` debe incluir las etiquetas de gobernanza obligatorias (`team`, `environment`, `cost-center`); de lo contrario, Gatekeeper rechazará su creación.

#### Observabilidad para Eventos de Seguridad

Los registros de auditoría de Keycloak registran intentos de inicio de sesión, emisión de tokens y acciones de usuario. Centralizar estos eventos de seguridad y correlacionarlos con la actividad del clúster resulta fundamental para distinguir fallos de configuración de accesos no autorizados.

#### Gestión Automatizada de Certificados TLS

Los certificados caducados causan interrupciones graves. La integración con **Let's Encrypt** [8] mediante **cert-manager** [7] gestiona automáticamente solicitudes, desafíos ACME, almacenamiento en secretos de Kubernetes y renovación automática 30 días antes del vencimiento.

> **Figura 3.3** - Ciclo de vida del certificado TLS con cert-manager

#### Redes de Confianza Cero y Políticas de Red

Mediante políticas de red y el modo estricto de mTLS en Istio, cada conexión entre servicios se autentica y cifra automáticamente con certificados rotados, impidiendo escuchas no autorizadas y ataques de intermediario (*man-in-the-middle*).

---

### Implementación del Principio de Mínimo Privilegio

Para configurar permisos granulares para canalizaciones de CI/CD:

1. Crea la cuenta de servicio con alcance de namespace (`service-account.yaml`):

```yaml
# service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-cd-deployer
  namespace: staging
  labels:
    app: ci-cd-pipeline
    environment: staging
```

2. Define el rol con los permisos mínimos necesarios (`role-minimal-deployer.yaml`):

```yaml
# role-minimal-deployer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: minimal-deployer
  namespace: staging
rules:
  # Read-only access to existing deployments
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
  # Update deployments (but not create/delete)
  - apiGroups: ["apps"]
    resources: ["deployments/scale", "deployments/status"]
    verbs: ["update", "patch"]
  # Manage pods for rollouts
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "delete"]
  # View logs for debugging
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]
  # Read ConfigMaps and Secrets (but not write)
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
```

3. Vincula el rol a la cuenta de servicio (`rolebinding.yaml`):

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-cd-deployer-binding
  namespace: staging
subjects:
  - kind: ServiceAccount
    name: ci-cd-deployer
    namespace: staging
roleRef:
  kind: Role
  name: minimal-deployer
  apiGroup: rbac.authorization.k8s.io
```

4. Para recursos de ámbito de clúster como registros de imágenes, utiliza un `ClusterRole` vinculado mediante un `RoleBinding` namespaced (`clusterrole-image-reader.yaml`):

```yaml
# clusterrole-image-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: image-registry-reader
rules:
  # Only read access to images/image streams
  - apiGroups: ["image.openshift.io"]
    resources: ["images", "imagestreams"]
    verbs: ["get", "list"]
```

5. Aplica y verifica las autorizaciones:

```bash
# Apply all configurations
kubectl apply -f service-account.yaml
kubectl apply -f role-minimal-deployer.yaml
kubectl apply -f rolebinding.yaml

# Verify the service account's permissions
kubectl auth can-i --as=system:serviceaccount:staging:ci-cd-deployer update deployments -n staging
# Output: yes

kubectl auth can-i --as=system:serviceaccount:staging:ci-cd-deployer create deployments -n staging
# Output: no (as intended - can update but not create)

kubectl auth can-i --as=system:serviceaccount:staging:ci-cd-deployer get pods -n production
# Output: no (cannot access other namespaces)
```

Antipatrón a evitar:

```yaml
# BAD: Overly permissive cluster-admin binding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ci-cd-admin # AVOID THIS
subjects:
  - kind: ServiceAccount
    name: ci-cd-deployer
    namespace: staging
roleRef:
  kind: ClusterRole
  name: cluster-admin # Way too permissive!
  apiGroup: rbac.authorization.k8s.io
```

---

### Registro de Auditoría y Cumplimiento

El registro exhaustivo de las acciones del servidor de la API (identidad del usuario, marca de tiempo y resultado) permite investigaciones de seguridad, informes de cumplimiento y seguimiento de cambios [9]. Los eventos críticos que requieren registro detallado incluyen:
- Decisiones de autenticación y autorización.
- Acceso a secretos.
- Modificaciones de RBAC.
- Operaciones privilegiadas.

Los registros de auditoría deben enviarse a plataformas de observabilidad externas con políticas de retención inmutables.

> **Ejercicio: Implementar seguridad de plataforma de extremo a extremo**
> 
> Configura la incorporación segura de un nuevo equipo llamado `"payments"` siguiendo 7 pasos:
> 1. Configuración de identidades y accesos en Keycloak (grupos y usuarios de prueba).
> 2. Configuración de RBAC y creación de namespaces para `payments`.
> 3. Creación y despliegue de políticas OPA (registros permitidos, límites de recursos y `PodDisruptionBudget`).
> 4. Implementación de seguridad de red (políticas de ingress/egress).
> 5. Configuración de certificados TLS y mTLS.
> 6. Validación de permisos con `kubectl auth can-i`.
> 7. Simulación de respuesta a incidentes (detección de token expuesto, revocación y auditoría).

---

### Resumen

Lecciones clave aprendidas con el equipo de María:
- **Defensa en profundidad**: Ninguna medida individual es suficiente; se deben superponer autenticación OAuth, autorización RBAC, control de admisión con OPA, políticas de red y cifrado mTLS.
- **Principio de mínimo privilegio**: Evita atajos como `cluster-admin`. Cada identidad debe disponer estrictamente de los permisos requeridos para su función.
- **Políticas como Código**: Mientras que RBAC controla *quién* despliega, OPA controla *qué* se despliega mediante reglas Rego declarativas.
- **Automatización**: Automatiza la gestión de certificados con cert-manager, la expiración de tokens con Keycloak y las comprobaciones de cumplimiento con OPA.
- **Federación de identidades**: Reduce la fricción integrando proveedores de identidad existentes en lugar de gestionar credenciales independientes de Kubernetes.
- **Observabilidad de seguridad**: Centraliza registros y eventos de seguridad para mantener trazabilidad completa.

La seguridad de la plataforma no busca restringir a los desarrolladores, sino permitirles realizar su trabajo de forma segura y confiable.

---

### Referencias

- **[1]** *Kubernetes OIDC Authentication*. [https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
- **[2]** *SOC 2 — AICPA Trust Services Criteria*. [https://www.aicpa-cima.com/resources/landing/soc-2](https://www.aicpa-cima.com/resources/landing/soc-2) / *HIPAA — HHS*. [https://www.hhs.gov/hipaa](https://www.hhs.gov/hipaa) / *PCI DSS — PCI Security Standards Council*. [https://www.pcisecuritystandards.org](https://www.pcisecuritystandards.org/)
- **[3]** *Keycloak — Open Source Identity and Access Management*. [https://www.keycloak.org/documentation](https://www.keycloak.org/documentation)
- **[4]** *CircleCI January 2023 Security Incident Report*. [https://circleci.com/blog/jan-4-2023-incident-report/](https://circleci.com/blog/jan-4-2023-incident-report/)
- **[5]** *OPA Gatekeeper — Kubernetes Policy Controller*. [https://open-policy-agent.github.io/gatekeeper/](https://open-policy-agent.github.io/gatekeeper/)
- **[6]** *Prometheus Operator — PrometheusRule CRD Reference*. [https://prometheus-operator.dev/docs/api-reference/api/#monitoring.coreos.com/v1.PrometheusRule](https://prometheus-operator.dev/docs/api-reference/api/#monitoring.coreos.com/v1.PrometheusRule)
- **[7]** *cert-manager — Certificate Management for Kubernetes*. [https://cert-manager.io/docs/](https://cert-manager.io/docs/)
- **[8]** *Let's Encrypt — ACME Protocol and Certificate Issuance*. [https://letsencrypt.org/docs/](https://letsencrypt.org/docs/)
- **[9]** *Kubernetes Audit Logging*. [https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/)
