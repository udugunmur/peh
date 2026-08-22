# Parte 2: Mejora de la Productividad a Través de Funciones de Autoservicio

## Capítulo 8: CI/CD como Servicio de Plataforma

En los capítulos anteriores desplegamos una aplicación de demostración, establecimos un portal de desarrolladores con Backstage y construimos capacidades de incorporación en modo autoservicio. Los desarrolladores de NewTech ahora pueden acceder a la plataforma, crear *namespaces* y desplegar aplicaciones. Sin embargo, existe un obstáculo: cada equipo está construyendo sus propias canalizaciones de CI/CD desde cero, lo que genera duplicación de esfuerzos, inconsistencias operativas y brechas de seguridad.

En este capítulo transformaremos CI/CD en un **servicio de plataforma** mediante la creación de canalizaciones reutilizables y componibles que cada equipo pueda ensamblar en un flujo estandarizado. El objetivo es proporcionar *Golden Paths* (caminos pavimentados): patrones de diseño con opiniones fundamentadas pero flexibles que codifiquen las mejores prácticas de la organización, permitiendo la personalización cuando sea necesario.

Al finalizar este capítulo, serás capaz de:
- Diseñar y publicar tareas de canalización reutilizables y versionadas.
- Implementar entrega progresiva (*progressive delivery*) mediante despliegues Canary y Blue-Green.
- Integrar observabilidad nativa en los flujos de CI/CD.
- Configurar mecanismos de reversión automática (*automated rollback*).
- Validar y probar las tareas de canalización mediante automatización en Python.
- Redesplegar la aplicación de demostración utilizando las nuevas tareas de plataforma.

---

### Contexto de CI/CD Reutilizable

A medida que crece la adopción de la plataforma en una organización, la cultura de "copiar y pegar" se propaga con rapidez. Si 50 equipos independientes mantienen entre 2 y 3 repositorios cada uno, se alcanzan fácilmente entre 100 y 150 configuraciones de canalización divergentes.

Problemas comunes de las canalizaciones duplicadas:
- **Postura de seguridad inconsistente**: Actualizar un escáner de vulnerabilidades obliga a modificar cientos de repositorios manualmente, aumentando el riesgo de omitir parches críticos.
- **Reinvención de la rueda**: Múltiples equipos invierten ciclos resolviendo problemas ya solventados (construcción de contenedores, inyección de secretos).
- **Desviación de cumplimiento (*compliance drift*)**: Las actualizaciones en las normativas de gobernanza no se propagan a todas las canalizaciones, generando hallazgos en las auditorías.
- **Dependencia del conocimiento informal (*tribal knowledge*)**: La rotación de personal dificulta el mantenimiento y la depuración de flujos fragmentados.

---

### La Visión de CI/CD de Plataforma

El modelo de propiedad se invierte: el equipo de plataforma mantiene una biblioteca de bloques de construcción modulares, probados y conformes con la normativa, mientras que los equipos de desarrollo componen sus propios flujos a partir de dichos bloques.

Principios clave de CI/CD como servicio de plataforma:
1. **Componibilidad sobre monolitos**: Tareas pequeñas y enfocadas que realizan una única función y pueden encadenarse.
2. **Versionado estricto**: Tareas, plantillas y políticas versionadas como artefactos semánticos (permitiendo fijar versiones estables).
3. **Pruebas rigurosas de plataforma**: Las tareas de CI/CD se someten a pruebas automatizadas antes de su lanzamiento como código de producción.
4. **Valores por defecto razonables con salidas de escape (*escape hatches*)**: Facilitar el camino estándar sin bloquear personalizaciones justificadas.
5. **Observabilidad por defecto**: Emisión de métricas, registros y trazas para supervisar la salud de las canalizaciones.

---

### Descripción General de la Arquitectura

> **Figura 8.1** - Arquitectura de CI/CD como servicio de plataforma

El modelo de tres capas establece una separación nítida de responsabilidades:
- **Capa Inferior (Equipo de Plataforma)**: Tareas reutilizables atómicas (compilación, escaneo de seguridad, pruebas).
- **Capa Intermedia (Plantillas de Plataforma)**: Plantillas compuestas por arquetipo (backend, frontend, ML, infraestructura).
- **Capa Superior (Equipos de Desarrollo)**: Repositorios de aplicación que invocan plantillas mediante configuraciones mínimas, entregando artefactos vía GitOps (Flux CD / Argo Rollouts) al clúster de Kubernetes.

#### Establecimiento de la Línea Base de Métricas

Para medir la reducción de duplicación y el avance de la adopción, el script `analyze_workflows.py` analiza los flujos de GitHub Actions en los repositorios de la organización:

```bash
python analyze_workflows.py /path/to/repos-root
```

Métricas clave de CI/CD y plataforma:

| Métrica | Qué Mide | Objetivo |
| :--- | :--- | :--- |
| **Frecuencia de despliegue (*Deployment frequency*)** | Con qué frecuencia llega el código a producción | Diaria o superior para equipos de alto rendimiento |
| **Tiempo de entrega de cambios (*Lead time for changes*)** | Tiempo desde el *commit* hasta producción | < 1 hora para equipos de alto rendimiento |
| **Tasa de fallos en cambios (*Change failure rate*)** | Porcentaje de despliegues que causan incidentes | < 5% |
| **Tiempo medio de recuperación (*MTTR*)** | Tiempo necesario para recuperarse de un fallo | < 1 hora |
| **Duración de compilación (*Build duration p95*)** | Tiempo de ejecución del percentil 95 de la canalización | Línea base × 1.5 como techo saludable |
| **Tasa de adopción de la plataforma (*Platform adoption rate*)** | Porcentaje de repositorios que usan acciones de plataforma | Seguimiento semana a semana; objetivo 100% |
| **Tasa de aprobación de escaneo de seguridad** | Porcentaje de compilaciones sin hallazgos CRÍTICOS | Tendencia hacia el 100% |

---

### Guías de Diseño de Canalizaciones (*Pipeline Playbooks*)

| Patrones que Escalan | Antipatrones a Evitar |
| :--- | :--- |
| Las canalizaciones de los equipos deben ser envoltorios mínimos sobre las plantillas de plataforma (típicamente menos de 30 líneas). | Un único archivo de flujo de trabajo de 2000 líneas que maneja todos los escenarios mediante lógica condicional compleja. |
| Tratar las acciones de plataforma como versiones de bibliotecas (semver, etiquetas flotantes `v1`, `v2`). | Canalizaciones que requieren que los ingenieros copien manualmente secretos desde un almacén a GitHub. |
| Gestión estricta de la configuración de CI/CD en Git para auditoría, reproducibilidad y reversión. | Ejecutores autohospedados (*self-hosted runners*) con dependencias no documentadas integradas en la imagen. |
| Instrumentación y observabilidad de canalizaciones antes de que surjan problemas de rendimiento. | Puertas de aprobación manual que solo existen para casillas de verificación de cumplimiento sin aportar valor real. |

#### Ejemplos de Golden Paths Operativos:
- **Actualizaciones de versiones de bases de datos**: Flujo que genera ramas de migración, ejecuta pruebas contra clones de base de datos y abre una PR con el script de reversión.
- **Incorporación a recuperación ante desastres (DR)**: Automatización que suscribe el servicio a manuales de DR y valida el RTO mediante simulacros.
- **Actualización automatizada de dependencias**: Flujo programado (Dependabot/Renovate) que agrupa parches y fusiona tras pasar la suite de pruebas.
- **Remediación de seguridad**: Flujo activado por alertas de Trivy que crea tickets y bloquea despliegues en producción ante vulnerabilidades no mitigadas.

---

### Creación de Tareas de Canalización Reutilizables

GitHub Actions ofrece dos mecanismos principales de reutilización:

| Aspecto | Acciones Compuestas (*Composite Actions*) | Flujos de Trabajo Reutilizables (*Reusable Workflows*) |
| :--- | :--- | :--- |
| **Alcance** | Pasos dentro de un único trabajo (*job*) | Trabajos completos con múltiples pasos |
| **Ejecutor (*Runner*)** | Hereda del trabajo que la invoca | Define su propio ejecutor |
| **Anidamiento** | Puede anidar hasta 10 niveles | No puede llamar a otros flujos de trabajo |
| **Secretos** | Entradas o variables de entorno | Secretos nativos |
| **Registro (*Logging*)** | Paso único colapsado | Visibilidad completa por trabajo y paso |
| **Ideal para** | Secuencias de pasos pequeñas y reutilizables | Patrones completos de canalización de CI/CD |

> **Tabla 8.1** - Acciones compuestas frente a flujos de trabajo reutilizables

> **Figura 8.2** - Estructura recomendada del repositorio de acciones de plataforma (`platform-actions`)

---

### Construcción de una Acción Compuesta (*Composite Action*)

Acción compuesta para construcción de contenedores con escaneo de seguridad integrado (`.github/actions/container-build/action.yml`):

```yaml
# .github/actions/container-build/action.yml
name: 'Platform Container Build'
description: 'Build and push container with security scanning'
inputs:
  image-name:
    description: 'Container image name'
    required: true
  dockerfile:
    description: 'Path to Dockerfile'
    default: 'Dockerfile'
  registry:
    description: 'Container registry URL'
    default: 'ghcr.io'
  context:
    description: 'Build context directory'
    default: '.'
outputs:
  image-digest:
    description: 'Image digest for verification'
    value: ${{ steps.build.outputs.digest }}
  image-tag:
    description: 'Full image tag'
    value: ${{ steps.meta.outputs.tags }}
runs:
  using: 'composite'
  steps:
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ inputs.registry }}/${{ inputs.image-name }}
    - name: Build and push
      id: build
      uses: docker/build-push-action@v6
      with:
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        file: ${{ inputs.dockerfile }}
        context: ${{ inputs.context }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    - name: Run Trivy vulnerability scan
      uses: aquasecurity/trivy-action@0.20.0
      with:
        image-ref: ${{ steps.meta.outputs.tags }}
        exit-code: '1'
        severity: 'CRITICAL,HIGH'
        format: 'sarif'
        output: 'trivy-results.sarif'
      shell: bash
```

> **Nota de Seguridad**: La acción compila la imagen localmente, ejecuta el escaneo con Trivy y solo publica en el registro si no se detectan vulnerabilidades críticas (`exit-code: '1'`).

#### Validación Automatizada de Acciones de Plataforma

Script en Python para validar la sintaxis y estructura de los archivos `action.yml`:

```python
#!/usr/bin/env python3
"""Validate GitHub Action configuration files."""
import sys
from pathlib import Path
import yaml

REQUIRED_FIELDS = ['name', 'description', 'runs']
VALID_USING_VALUES = ['composite', 'node16', 'node20', 'docker']

def validate_action(action_path: Path) -> list[str]:
    """Validate a single action.yml file. Returns list of errors."""
    errors = []
    try:
        content = yaml.safe_load(action_path.read_text())
    except yaml.YAMLError as e:
        return [f"YAML parse error: {e}"]

    # Check required fields
    for field in REQUIRED_FIELDS:
        if field not in content:
            errors.append(f"Missing required field: {field}")

    # Validate 'using' value
    if 'runs' in content:
        using = content['runs'].get('using', '')
        if using not in VALID_USING_VALUES:
            errors.append(f"Invalid using value: {using}")

    # Check composite action has steps
    if using == 'composite' and 'steps' not in content['runs']:
        errors.append("Composite action missing 'steps'")

    # Validate inputs have descriptions
    for name, config in content.get('inputs', {}).items():
        if 'description' not in config:
            errors.append(f"Input {name} missing description")

    return errors

def main():
    actions_dir = Path(".github/actions")
    exit_code = 0
    for action_file in actions_dir.glob("*/action.yml"):
        errors = validate_action(action_file)
        if errors:
            print(f"❌ {action_file}: {len(errors)} errors")
            for error in errors:
                print(f"  - {error}")
            exit_code = 1
        else:
            print(f"✅ {action_file}: Valid")
    sys.exit(exit_code)

if __name__ == "__main__":
    main()
```

---

### Desarrollo de Plantillas de Canalización

> **Figura 8.3** - Patrón de composición de canalizaciones

Arquetipos estándar de plantillas:
1. Microservicios backend.
2. Aplicaciones frontend.
3. Módulos de infraestructura.
4. Canalizaciones de ML.
5. Paquetes y bibliotecas compartidas.

#### Plantilla de Flujo de Trabajo Reutilizable (`.github/workflows/backend-pipeline.yml`)

```yaml
# .github/workflows/backend-pipeline.yml
name: Backend Microservice Pipeline
on:
  workflow_call:
    inputs:
      service-name:
        description: 'Name of the service being built'
        required: true
        type: string
      node-version:
        description: 'Node.js version to use'
        required: false
        type: string
        default: '20'
      deploy-environment:
        description: 'Target deployment environment'
        required: true
        type: string
      skip-tests:
        description: 'Skip test execution (not recommended)'
        required: false
        type: boolean
        default: false
    secrets:
      registry-token:
        description: 'Container registry authentication token'
        required: true
    outputs:
      image-tag:
        description: 'Deployed image tag'
        value: ${{ jobs.build.outputs.image-tag }}
      deployment-url:
        description: 'URL of deployed service'
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  test:
    if: ${{ !inputs.skip-tests }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
      - run: npm run lint

  build:
    needs: [test]
    if: always() && (needs.test.result == 'success' || inputs.skip-tests)
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.container.outputs.image-tag }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.registry-token }}
      # When calling from the platform repo itself (e.g., in templates):
      - uses: ./.github/actions/container-build@v1
        with:
          image-name: ${{ inputs.service-name }}
      # When calling from a team's application repo:
      - uses: platform-org/platform-actions/.github/actions/container-build@v1
        with:
          image-name: ${{ inputs.service-name }}
```

#### Invocación de la Plantilla desde el Repositorio del Equipo (`.github/workflows/ci.yml`)

```yaml
# Team repository: .github/workflows/ci.yml
name: CI Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  pipeline:
    uses: platform-org/platform-actions/.github/workflows/backend-pipeline.yml@v1
    with:
      service-name: order-service
      node-version: '20'
      deploy-environment: staging
    secrets:
      registry-token: ${{ secrets.GHCR_TOKEN }}
```

---

### Estrategia de Gestión de Versiones

Automatización de etiquetado semántico y actualización de etiquetas principales flotantes (`v1`, `v2`):

```python
#!/usr/bin/env python3
"""Manage semantic version tags for platform actions."""
import subprocess
import re
from dataclasses import dataclass

@dataclass
class SemVer:
    major: int
    minor: int
    patch: int

    def __str__(self) -> str:
        return f"v{self.major}.{self.minor}.{self.patch}"

    def bump_major(self) -> 'SemVer':
        return SemVer(self.major + 1, 0, 0)

    def bump_minor(self) -> 'SemVer':
        return SemVer(self.major, self.minor + 1, 0)

    def bump_patch(self) -> 'SemVer':
        return SemVer(self.major, self.minor, self.patch + 1)

def get_latest_tag() -> SemVer:
    """Get the latest semantic version tag from git."""
    result = subprocess.run(
        ['git', 'tag', '-l', 'v*', '--sort=-v:refname'],
        capture_output=True,
        text=True
    )
    tags = result.stdout.strip().split('\n')
    if not tags or not tags[0]:
        return SemVer(0, 0, 0)
    match = re.match(r'v(\d+)\.(\d+)\.(\d+)', tags[0])
    if match:
        return SemVer(int(match[1]), int(match[2]), int(match[3]))
    return SemVer(0, 0, 0)

def create_tags(version: SemVer) -> None:
    """Create version tag and update floating major tag."""
    # Create specific version tag
    subprocess.run(['git', 'tag', str(version)])
    # Update floating major tag (e.g., v1 points to latest v1.x.x)
    major_tag = f"v{version.major}"
    subprocess.run(['git', 'tag', '-f', major_tag])
    subprocess.run(['git', 'push', 'origin', str(version)])
    subprocess.run(['git', 'push', 'origin', '-f', major_tag])
```

---

### Entrega Progresiva y Reversión Automática (*Progressive Delivery & Rollbacks*)

A diferencia de los despliegues binarios tradicionales, la entrega progresiva mitiga el radio de impacto (*blast radius*) desviando el tráfico de forma gradual mientras evalúa métricas en tiempo real mediante **Argo Rollouts** [2].

#### Estrategia Blue-Green
Mantiene dos entornos completos (Blue = Activo, Green = Previsualización). Tras la validación en Green, el balanceador de carga redirige el 100% del tráfico instantáneamente.

> **Figura 8.4** - Flujo de despliegue Blue-Green

#### Estrategia Canary
Desvía porcentajes incrementales de tráfico (10%, 25%, 50%, 100%) evaluando consultas a Prometheus (`AnalysisTemplate`). Si la tasa de éxito cae por debajo del 95%, Argo Rollouts revierte automáticamente al estado estable sin intervención humana.

> **Figura 8.5** - Flujo de despliegue Canary

---

### Observabilidad en las Canalizaciones de CI/CD

Configuración del recolector de OpenTelemetry para procesar webhooks de GitHub Actions y convertirlos en trazas distribuidas:

```yaml
receivers:
  github:
    webhook:
      endpoint: 0.0.0.0:19418
      path: /events
      health_path: /health
      secret: ${env:GITHUB_WEBHOOK_SECRET}
    scrapers:
      scraper: {} # required even if empty; enables the default GitHub scraper
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    metrics:
      receivers: [github]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      exporters: [otlp] # forward to your tracing backend
```

---

### Ejercicio 8.1: Despliegue de la Aplicación con Canalizaciones Compuestas

Implementa un flujo de entrega estandarizado para la aplicación de demostración:
1. Crear la estructura del repositorio `platform-actions`.
2. Añadir la acción compuesta de construcción de contenedores (`container-build`).
3. Crear la plantilla reutilizable de backend (`backend-pipeline.yml`).
4. Validar las acciones con el script de pruebas y generar etiquetas semánticas (`v1`).
5. Actualizar la canalización de la aplicación de demostración.
6. Instalar Argo Rollouts y configurar el despliegue Canary con métricas de análisis.
7. Verificar las trazas de ejecución de la canalización en Grafana.

---

### Resumen

- **Inversión del modelo de propiedad**: El equipo de plataforma provee bloques modulares versionados y probados; los equipos de desarrollo los componen en canalizaciones mínimas (menos de 30 líneas).
- **Acciones compuestas y flujos reutilizables**: Encapsulan pasos atómicos y patrones completos de ejecución, respectivamente.
- **Gestión de versiones con etiquetas flotantes**: Las etiquetas `v1` y `v2` garantizan actualizaciones no disruptivas manteniendo estabilidad.
- **Entrega progresiva con Argo Rollouts**: Reduce el riesgo en producción mediante despliegues Canary basados en análisis automatizado de métricas de Prometheus.
- **Observabilidad en CI/CD**: Transforma eventos de canalización en trazas de OpenTelemetry para monitorizar cuellos de botella y métricas DORA.

---

### Referencias

- **[1]** *GitHub Actions Documentation — Reusing Workflows and Composite Actions*. [https://docs.github.com/en/actions](https://docs.github.com/en/actions)
- **[2]** *Argo Rollouts — Progressive Delivery for Kubernetes*. [https://argoproj.github.io/rollouts/](https://argoproj.github.io/rollouts/)
- **[3]** *OpenTelemetry CI/CD Observability (CNCF Blog)*. [https://www.cncf.io/blog/2024/11/04/opentelemetry-is-expanding-into-ci-cd-observability/](https://www.cncf.io/blog/2024/11/04/opentelemetry-is-expanding-into-ci-cd-observability/)
