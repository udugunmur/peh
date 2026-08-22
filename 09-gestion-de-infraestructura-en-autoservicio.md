# Parte 2: Mejora de la Productividad a Través de Funciones de Autoservicio

## Capítulo 9: Gestión de Infraestructura en Autoservicio

En los capítulos anteriores establecimos canalizaciones de CI/CD para compilar y desplegar aplicaciones utilizando las distintas capacidades de la plataforma. La agilidad del equipo de María para incorporar desarrolladores, aprovisionar *namespaces* y desplegar servicios en contenedores ha aumentado sustancialmente. Sin embargo, persiste una fricción: ¿qué ocurre cuando un equipo necesita recursos que van más allá del cómputo? Los desarrolladores continúan abriendo solicitudes (*tickets*) para aprovisionar bases de datos PostgreSQL, cachés de Redis y nodos con GPU para entrenamiento de modelos de ML.

En este capítulo abordaremos el aprovisionamiento de infraestructura como una **capacidad de autoservicio**. Desplegaremos **Crossplane** [1] para habilitar la gestión declarativa de infraestructura, diseñaremos planos maestros compuestos (*composite blueprints*), estableceremos configuraciones gobernadas y aprovisionaremos una base de datos conectada a la aplicación de demostración.

---

### ¿Por qué Necesitamos Autoservicio de Infraestructura en la Plataforma?

El modelo tradicional de aprovisionamiento mediante tickets trata cada solicitud como un caso artesanal:
- **Cuello de botella operativo**: Todas las solicitudes se canalizan a través de un equipo reducido de operaciones o plataforma.
- **Competencia desordenada por prioridades**: Solicitudes triviales compiten con necesidades críticas, incentivando a los desarrolladores a manipular campos de prioridad.
- **Desviación de configuración (*environment drift*)**: La falta de visibilidad sobre la intención exacta del desarrollador genera discrepancias entre entornos.

Con el autoservicio declarativo, los desarrolladores definen sus requisitos en archivos YAML versionados en Git junto al código fuente de su aplicación. La plataforma interpreta estas declaraciones, aplica políticas y salvaguardas (*guardrails*), aprovisiona los recursos en la nube mediante controladores de reconciliación continua e inyecta las credenciales de conexión directamente en el clúster.

---

### Catálogos de Servicios frente a Frameworks de Infraestructura como Código

Crossplane traslada el modelo declarativo de Kubernetes a la infraestructura externa:
- **Infraestructura versionada junto a la aplicación**: Reside en el mismo repositorio Git, sujeta al mismo flujo de revisión de código (*Pull Requests*) y auditoría.
- **Inyección automatizada de secretos**: Elimina el intercambio manual de credenciales entre equipos de infraestructura y desarrollo.
- **Reconciliación continua**: Si la configuración en la nube difiere de la declaración en Git, los controladores corrigen la desviación (*drift*).

> **Consideración Práctica**: Crossplane es idóneo para organizaciones con inversión previa en Kubernetes y múltiples equipos solicitando patrones de infraestructura recurrentes. Para equipos pequeños con cambios esporádicos, módulos de Terraform con flujos sencillos de CI pueden resultar más rentables operativamente.

---

### Planos Maestros de Infraestructura (*Infrastructure Blueprints*)

> **Figura 9.1** - Arquitectura de tres capas de Crossplane

La arquitectura de Crossplane se organiza en tres niveles:
1. **Proveedores (*Providers*)**: Integraciones con APIs externas (AWS, Azure, GCP o Kubernetes).
2. **Recursos Gestionados (*Managed Resources - MR*)**: Representaciones individuales de componentes de infraestructura (ej. instancia RDS, bucket S3).
3. **Recursos Compuestos (*Composite Resources - XR*)**: Abstracciones de alto nivel que combinan múltiples recursos gestionados ocultando la complejidad subyacente.

> **Figura 9.2** - Flujo de aprovisionamiento de extremo a extremo desde el *Claim* hasta la aplicación

#### Configuración de Proveedores en Crossplane (`crossplane-providers.yaml`)

```yaml
# crossplane-providers.yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-family-aws:v1.7.0
  runtimeConfigRef:
    name: aws-config
---
apiVersion: pkg.crossplane.io/v1beta1
kind: ControllerConfig
metadata:
  name: aws-config
spec:
  resources:
    limits:
      memory: 512Mi
      cpu: 500m
    requests:
      memory: 256Mi
      cpu: 100m
---
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-credentials
      key: credentials
```

---

### Diseño de Recursos Compuestos (*Composite Resources*)

La composición permite exponer únicamente los parámetros relevantes para los desarrolladores, codificando estándares organizacionales (rangos de almacenamiento, versiones permitidas y políticas de copias de seguridad).

#### 1. Definición del Recurso Compuesto (`xrd-postgresql.yaml`)

```yaml
# xrd-postgresql.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: postgresqlinstances.database.platform.io
spec:
  group: database.platform.io
  names:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
    claimNames:
      kind: PostgreSQLClaim
      plural: postgresqlclaims
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    storageGB:
                      type: integer
                      description: Storage size in GB
                      default: 20
                      minimum: 15
                      maximum: 500
                    version:
                      type: string
                      description: PostgreSQL version
                      default: "15"
                      enum: ["13", "14", "15", "16"]
                    tier:
                      type: string
                      description: Performance tier
                      default: development
                      enum: [development, staging, production]
                    enableBackups:
                      type: boolean
                      description: Enable automated backups
                      default: true
                  required:
                    - tier
              required:
                - parameters
            status:
              type: object
              properties:
                connectionSecret:
                  type: string
                  description: Name of secret containing connection details
                endpoint:
                  type: string
                  description: Database endpoint
                port:
                  type: integer
                  description: Database port
```

#### 2. Composición de Infraestructura (`composition-postgresql.yaml`)

Mapea los parámetros de alto nivel del *Claim* a las configuraciones de AWS RDS según el nivel de rendimiento (*tier*):

```yaml
# composition-postgresql.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: postgresql-aws
  labels:
    provider: aws
    database: postgresql
spec:
  compositeTypeRef:
    apiVersion: database.platform.io/v1alpha1
    kind: PostgreSQLInstance
  writeConnectionSecretsToNamespace: crossplane-system
  patchSets:
    - name: common-labels
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: metadata.labels
          toFieldPath: metadata.labels
          policy:
            mergeOptions:
              keepMapValues: true
    - name: tier-settings
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.tier
          toFieldPath: spec.forProvider.instanceClass
          transforms:
            - type: map
              map:
                development: db.t3.micro
                staging: db.t3.small
                production: db.r6g.large
  resources:
    - name: rds-instance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            engine: postgres
            publiclyAccessible: false
            storageEncrypted: true
            autoMinorVersionUpgrade: true
            deletionProtection: false
            skipFinalSnapshot: true
            vpcSecurityGroupIdSelector:
              matchLabels:
                platform.io/resource: database-sg
            dbSubnetGroupNameSelector:
              matchLabels:
                platform.io/resource: database-subnet
            writeConnectionSecretToRef:
              namespace: crossplane-system
      patches:
        - type: PatchSet
          patchSetName: common-labels
        - type: PatchSet
          patchSetName: tier-settings
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.storageGB
          toFieldPath: spec.forProvider.allocatedStorage
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.version
          toFieldPath: spec.forProvider.engineVersion
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.enableBackups
          toFieldPath: spec.forProvider.backupRetentionPeriod
          transforms:
            - type: map
              map:
                "true": 7
                "false": 0
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.endpoint
          toFieldPath: status.endpoint
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.port
          toFieldPath: status.port
      connectionDetails:
        - name: endpoint
          fromFieldPath: status.atProvider.endpoint
        - name: port
          fromFieldPath: status.atProvider.port
        - name: username
          fromFieldPath: spec.forProvider.username
        - name: password
          fromConnectionSecretKey: attribute.password
```

---

### Gobernanza con Salvaguardas y Etiquetado (*Guardrails & Tagging*)

El autoservicio requiere gobernanza automatizada para evitar costes descontrolados y vulnerabilidades de seguridad.

#### 1. Conjunto de Parches de Etiquetado Obligatorio (`tagging-patchset.yaml`)

```yaml
# tagging-patchset.yaml
patchSets:
  - name: required-tags
    patches:
      - type: FromCompositeFieldPath
        fromFieldPath: metadata.labels["platform.io/team"]
        toFieldPath: spec.forProvider.tags["Team"]
      - type: FromCompositeFieldPath
        fromFieldPath: metadata.labels["platform.io/cost-center"]
        toFieldPath: spec.forProvider.tags["CostCenter"]
      - type: FromCompositeFieldPath
        fromFieldPath: spec.parameters.tier
        toFieldPath: spec.forProvider.tags["Environment"]
      - type: CombineFromComposite
        combine:
          variables:
            - fromFieldPath: metadata.namespace
            - fromFieldPath: metadata.name
          strategy: string
          string:
            fmt: "%s/%s"
        toFieldPath: spec.forProvider.tags["ManagedBy"]
```

#### 2. Webhook de Admisión para Validación de Políticas (`guardrail-validator.py`)

> **Figura 9.3** - Flujo de decisión de salvaguardas mediante Webhook de Admisión de Validación

```python
# guardrail-validator.py
#!/usr/bin/env python3
"""# NOTE: We check tier before namespace because the error message is clearer that way."""
from flask import Flask, request, jsonify
import json

app = Flask(__name__)

PRODUCTION_NAMESPACES = {"production", "prod", "prd"}
TIER_NAMESPACE_RULES = {
    "production": PRODUCTION_NAMESPACES,
    "staging": {"staging", "stg", "qa"} | PRODUCTION_NAMESPACES,
    "development": None  # Any namespace allowed
}

@app.route("/validate", methods=["POST"])
def validate():
    """Validate admission request against policies."""
    admission_review = request.get_json()
    req = admission_review["request"]
    obj = req["object"]
    namespace = req["namespace"]

    tier = obj.get("spec", {}).get("parameters", {}).get("tier", "development")
    allowed_namespaces = TIER_NAMESPACE_RULES.get(tier)

    if allowed_namespaces is not None and namespace not in allowed_namespaces:
        return jsonify({
            "apiVersion": "admission.k8s.io/v1",
            "kind": "AdmissionReview",
            "response": {
                "uid": req["uid"],
                "allowed": False,
                "status": {
                    "code": 403,
                    "message": f"Tier '{tier}' is not allowed in namespace '{namespace}'. "
                               f"Production resources must be in: {PRODUCTION_NAMESPACES}"
                }
            }
        })

    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": req["uid"],
            "allowed": True
        }
    })

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8443, ssl_context="adhoc")
```

---

### Autonomía de Infraestructura: ¿Libertad o Caos?

La verdadera autonomía no consiste en delegar configuraciones de bajo nivel (VPCs, grupos de seguridad), sino en ofrecer componentes listos para el consumo que apliquen las normas corporativas de forma invisible.

Principios de éxito:
1. **Hacer que lo correcto sea lo más fácil**: Planos maestros preconfigurados como la ruta de menor resistencia.
2. **Hacer visible lo incorrecto**: Etiquetado exhaustivo, seguimiento de costes y auditorías reflejadas en paneles de control.
3. **Mecanismos de escape explícitos**: Procesos claros para solicitar recursos especializados con justificación de negocio.

---

### Ejercicio 9.1: Generación de Composiciones Específicas por Entorno

Generar automáticamente composiciones para Desarrollo, Staging y Producción a partir de una única estructura de datos en Python para evitar discrepancias de configuración:
- **Desarrollo**: `db.t3.micro`, retención mínima de copias de seguridad.
- **Staging**: `db.t3.small`, retención de 7 días, *Performance Insights*.
- **Producción**: `db.r6g.large`, despliegue Multi-AZ, retención de 30 días y protección contra eliminación.

---

### Automatización del Ciclo de Vida de Recursos

Controlador en Python para monitorizar límites de antigüedad y etiquetas obligatorias (`lifecycle-controller.py`):

```python
# lifecycle-controller.py
#!/usr/bin/env python3
"""Infrastructure lifecycle controller - enforces age limits and ownership policies."""
import asyncio
from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import Optional
from kubernetes import client, config, watch

@dataclass
class LifecyclePolicy:
    max_age_days: Optional[int] = None
    require_owner_label: bool = True
    auto_cleanup: bool = False

POLICIES = {
    "development": LifecyclePolicy(max_age_days=30, auto_cleanup=True),
    "staging": LifecyclePolicy(max_age_days=90),
    "production": LifecyclePolicy(max_age_days=None, require_owner_label=True)
}

class LifecycleController:
    def __init__(self):
        config.load_incluster_config()
        self.api = client.CustomObjectsApi()

    def check_violations(self, resource: dict) -> list[str]:
        tier = resource.get("spec", {}).get("parameters", {}).get("tier", "development")
        policy = POLICIES.get(tier, POLICIES["development"])
        violations = []

        # Check age limit
        if policy.max_age_days:
            created = datetime.fromisoformat(
                resource["metadata"]["creationTimestamp"].replace("Z", "+00:00"))
            age = (datetime.now(created.tzinfo) - created).days
            if age > policy.max_age_days:
                violations.append(f"Age {age}d exceeds {policy.max_age_days}d limit")

        # Check owner label
        if policy.require_owner_label:
            if "platform.io/owner" not in resource.get("metadata", {}).get("labels", {}):
                violations.append("Missing platform.io/owner label")

        return violations

    async def run(self):
        for event in watch.Watch().stream(
            self.api.list_cluster_custom_object,
            group="database.platform.io",
            version="v1alpha1",
            plural="postgresqlclaims"
        ):
            if event["type"] in ["ADDED", "MODIFIED"]:
                violations = self.check_violations(event["object"])
                if violations:
                    name = event["object"]["metadata"]["name"]
                    print(f"Policy violations for {name}: {violations}")

if __name__ == "__main__":
    asyncio.run(LifecycleController().run())
```

---

### Habilitación de Innovación: Planos Maestros de GPU (`xrd-gpu-nodepool.yaml`)

Definición para aprovisionamiento de nodos GPU dedicados a cargas de trabajo de ML:

```yaml
# xrd-gpu-nodepool.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: gpunodepools.compute.platform.io
spec:
  group: compute.platform.io
  names:
    kind: GPUNodePool
    plural: gpunodepools
    claimNames:
      kind: GPUNodePoolClaim
      plural: gpunodepoolclaims
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    gpuType:
                      type: string
                      description: GPU type
                      enum: [nvidia-t4, nvidia-a10g, nvidia-a100]
                      default: nvidia-t4
                    nodeCount:
                      type: integer
                      description: Number of GPU nodes
                      minimum: 1
                      maximum: 10
                      default: 1
                    spotInstances:
                      type: boolean
                      description: Use spot instances for cost savings
                      default: true
                    schedulingWindow:
                      type: object
                      description: When nodes should be available
                      properties:
                        startHour:
                          type: integer
                          minimum: 0
                          maximum: 23
                        endHour:
                          type: integer
                          minimum: 0
                          maximum: 23
                        timezone:
                          type: string
                          default: UTC
                  required:
                    - gpuType
```

---

### Integración de Infraestructura en la Aplicación de Demostración

#### 1. Solicitud de Base de Datos (`demo-app/infrastructure/database.yaml`)

```yaml
# demo-app/infrastructure/database.yaml
apiVersion: database.platform.io/v1alpha1
kind: PostgreSQLClaim
metadata:
  name: demo-app-db
  namespace: team-alpha
  labels:
    platform.io/team: team-alpha
    platform.io/owner: demo-app
    platform.io/cost-center: engineering
spec:
  parameters:
    storageGB: 20
    version: "15"
    tier: development
    enableBackups: true
  writeConnectionSecretToRef:
    name: demo-app-db-connection
```

#### 2. Consumo de Credenciales en el Despliegue (`demo-app/deployment.yaml`)

```yaml
# demo-app/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: team-alpha
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: app
          image: ghcr.io/platform-org/demo-app:v1.2.0
          ports:
            - containerPort: 8080
          env:
            - name: DATABASE_HOST
              valueFrom:
                secretKeyRef:
                  name: demo-app-db-connection
                  key: endpoint
            - name: DATABASE_PORT
              valueFrom:
                secretKeyRef:
                  name: demo-app-db-connection
                  key: port
            - name: DATABASE_USER
              valueFrom:
                secretKeyRef:
                  name: demo-app-db-connection
                  key: username
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: demo-app-db-connection
                  key: password
            - name: DATABASE_NAME
              value: demo_app
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

#### 3. Validación Automatizada de Aprovisionamiento y Conectividad (`test-infrastructure.py`)

```python
# test-infrastructure.py
#!/usr/bin/env python3
"""Test infrastructure provisioning and connectivity."""
import subprocess
import time
import sys
from typing import Optional

def run_kubectl(args: list[str]) -> tuple[int, str, str]:
    """Run kubectl command and return exit code, stdout, stderr."""
    result = subprocess.run(
        ["kubectl"] + args,
        capture_output=True,
        text=True
    )
    return result.returncode, result.stdout, result.stderr

def wait_for_claim_ready(
    name: str,
    namespace: str,
    timeout: int = 300
) -> bool:
    """Wait for PostgreSQL claim to become ready."""
    start_time = time.time()
    while time.time() - start_time < timeout:
        code, stdout, _ = run_kubectl([
            "get", "postgresqlclaim", name,
            "-n", namespace,
            "-o", "jsonpath={.status.conditions[?(@.type=='Ready')].status}"
        ])
        if code == 0 and stdout.strip() == "True":
            return True
        print(f"Waiting for claim {name} to be ready...")
        time.sleep(10)
    return False

def verify_secret_exists(name: str, namespace: str) -> bool:
    """Verify connection secret was created."""
    code, _, _ = run_kubectl([
        "get", "secret", name,
        "-n", namespace
    ])
    return code == 0

def verify_app_connectivity(namespace: str) -> bool:
    """Verify application can connect to database."""
    code, stdout, _ = run_kubectl([
        "exec", "-n", namespace,
        "deployment/demo-app", "--",
        "curl", "-s", "localhost:8080/health"
    ])
    if code != 0:
        return False
    return "database" in stdout.lower() and "connected" in stdout.lower()

def main():
    namespace = "team-alpha"
    claim_name = "demo-app-db"
    secret_name = "demo-app-db-connection"

    print("Testing infrastructure provisioning...")

    # Apply the claim
    code, _, stderr = run_kubectl([
        "apply", "-f", "demo-app/infrastructure/database.yaml"
    ])
    if code != 0:
        print(f"Failed to apply claim: {stderr}")
        sys.exit(1)

    # Wait for claim to be ready
    if not wait_for_claim_ready(claim_name, namespace):
        print("Claim did not become ready in time")
        sys.exit(1)
    print("Claim is ready")

    # Verify secret exists
    if not verify_secret_exists(secret_name, namespace):
        print("Connection secret was not created")
        sys.exit(1)
    print("Connection secret exists")

    # Verify application connectivity
    if not verify_app_connectivity(namespace):
        print("Application cannot connect to database")
        sys.exit(1)
    print("Application connected to database successfully")

    print("All infrastructure tests passed!")

if __name__ == "__main__":
    main()
```

---

### Ejercicio 9.2: Despliegue de Infraestructura con Crossplane

1. Instalar Crossplane mediante el chart de Helm correspondiente.
2. Desplegar el proveedor de Kubernetes para pruebas locales.
3. Aplicar la definición XRD y la composición de PostgreSQL.
4. Crear la solicitud de base de datos (`PostgreSQLClaim`) para la aplicación de demostración.
5. Actualizar el manifiesto de despliegue para inyectar las variables de entorno desde el secreto generado.
6. Ejecutar `test-infrastructure.py` para validar la conectividad de la base de datos.
7. Desplegar el controlador de ciclo de vida y evaluar las políticas de gobernanza.

---

### Resumen

- **Infraestructura declarativa en autoservicio**: Permite a los desarrolladores solicitar recursos externos mediante manifiestos YAML versionados junto a su código fuente.
- **Abstracción con Crossplane**: Oculta la complejidad de los proveedores de nube mediante recursos compuestos (XRDs y Compositions).
- **Gobernanza automatizada**: Aplica etiquetas obligatorias y políticas de admisión mediante webhooks de validación.
- **Inyección de secretos de conexión**: Conecta automáticamente las aplicaciones con sus dependencias de infraestructura a través de secretos de Kubernetes.
- **Gestión del ciclo de vida**: Supervisa límites de antigüedad, eliminación de recursos huérfanos y costes operativos.

---

### Referencias

- **[1]** *Crossplane Documentation*. [https://crossplane.io/docs](https://crossplane.io/docs)
- **[2]** *Upbound Crossplane Providers*. [https://docs.upbound.io/providers/](https://docs.upbound.io/providers/)
- **[3]** *Kubernetes Custom Resources Concepts*. [https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
