# Parte 2: Mejora de la Productividad a Través de Funciones de Autoservicio

## Capítulo 10: Publicación de Starter Kits

En los capítulos anteriores construimos los componentes necesarios para desplegar aplicaciones de pila completa (*full-stack*): canalizaciones de CI/CD que componen tareas reutilizables, planos maestros de infraestructura que aprovisionan bases de datos y cachés, y un portal de desarrolladores que ofrece una interfaz unificada. Sin embargo, surge un nuevo desafío.

Cada vez que un equipo inicia un nuevo microservicio, dedica las dos primeras semanas a copiar código de proyectos existentes, adaptar configuraciones y depurar canalizaciones de CI/CD. Copiar proyectos desactualizados propaga versiones obsoletas, parches de seguridad faltantes y código prototipo no apto para producción. Además, publicar plantillas con errores no detectados afecta a múltiples equipos simultáneamente, convirtiendo la mejora de productividad prometida en un desperdicio de tiempo.

En este capítulo resolveremos estos problemas: diseñaremos una arquitectura de plantillas que equilibre estandarización y personalización, construiremos andamiajes de proyectos utilizando el **Scaffolder de Backstage**, configuraremos mecanismos de seguimiento de actualizaciones con Renovate, publicaremos las plantillas en el catálogo del portal y validaremos el flujo integral de creación de servicios.

---

### ¿Cuándo Tienen Sentido los Starter Kits?

Los *starter kits* implican un coste continuo de mantenimiento (actualización de dependencias, corrección de errores y alineación con las nuevas capacidades de la plataforma).

#### Cuándo Desarrollar Starter Kits:
- Más de 3 o 4 equipos creando proyectos basados en arquitecturas similares.
- Múltiples versiones de productos que requieren fundamentos técnicos homogéneos.
- Incidentes recurrentes originados por la desviación de configuraciones copiadas y pegadas.

#### Cuándo Evitarlos:
- Proyectos experimentales o arquitecturas genuinamente novedosas que no encajan en los estándares actuales.
- Equipos reducidos (1 o 2 equipos) con un único producto, donde un repositorio de ejemplo bien documentado es suficiente.

---

### Revisión de la Arquitectura de Plantillas

Un *starter kit* no es solo una colección de código repetitivo (*boilerplate*): codifica la forma en que deben construirse las aplicaciones en la organización (estructura de proyecto, pruebas, CI/CD, observabilidad, seguridad y empaquetado).

> **Figura 10.1** - Arquitectura de un Starter Kit eficaz (Capas de Plantilla, Andamiaje y Distribución)

#### Estructura del Repositorio de Starter Kits (`starter-kits/`)

```text
starter-kits/
├── templates/
│   ├── backend-service/
│   │   ├── v1/
│   │   │   ├── template/              # Actual project files
│   │   │   │   ├── src/
│   │   │   │   ├── tests/
│   │   │   │   ├── Dockerfile
│   │   │   │   ├── package.json
│   │   │   │   └── README.md
│   │   │   ├── generator/             # Yeoman generator
│   │   │   │   ├── index.js
│   │   │   │   └── prompts.js
│   │   │   └── backstage/             # Portal integration
│   │   │       └── template.yaml
│   │   └── v2/                        # ... newer version
│   ├── frontend-app/
│   │   └── v1/
│   │       ├── template/
│   │       ├── generator/
│   │       └── backstage/
│   └── data-pipeline/
│       └── v1/
│           ├── template/
│           ├── generator/
│           └── backstage/
├── shared/
│   ├── common-files/                  # Files shared across templates
│   └── helpers/                       # Template helper functions
├── scripts/
│   ├── publish.py                     # Publish to Backstage
│   ├── validate.py                    # Template validation
│   └── upgrade.py                     # Help projects upgrade
└── tests/
    └── template-tests/                # Tests for each template
```

---

### ¿Por qué Utilizamos el Scaffolder de Backstage?

Razones clave para elegir el Scaffolder integrado de Backstage frente a generadores locales externos:
1. **Elimina dependencias locales**: Los desarrolladores solo requieren un navegador web (sin instalaciones globales de paquetes ni discrepancias de versiones de Node/Python).
2. **Integración nativa con la identidad organizacional**: Conoce al creador, su equipo y sus permisos, inyectando metadatos de propiedad, RBAC y centro de costes automáticamente.
3. **Interfaz guiada y consistente**: Accesible para desarrolladores de cualquier perfil y nivel de experiencia.

> **Uso de Yeoman o Herramientas CLI**: Útiles para experimentación en entornos locales sin infraestructura de portal o para automatización en scripts cuando no se dispone de acceso a la API del portal.

---

### Andamiaje con Plantillas de Backstage

Definición de plantilla para servicio backend Node.js (`templates/backend-service/v1/template.yaml`):

```yaml
# templates/backend-service/v1/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: backend-service-v1
  title: Backend Service
  description: Create a new Node.js backend service with platform integrations
  tags:
    - nodejs
    - backend
    - recommended
  annotations:
    backstage.io/managed-by-location: url:https://github.com/platform-org/starter-kits
spec:
  owner: platform-team
  type: service
  parameters:
    - title: Service Information
      required:
        - serviceName
        - team
      properties:
        serviceName:
          title: Service Name
          type: string
          description: Unique name for the service (lowercase, alphanumeric, hyphens)
          pattern: "^[a-z][a-z0-9-]*$"
          ui:autofocus: true
        team:
          title: Team
          type: string
          description: The team that owns this service
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group
        description:
          title: Description
          type: string
          description: A brief description of the service
    - title: Configuration
      properties:
        database:
          title: Database
          type: string
          description: Database requirement for the service
          default: none
          enum:
            - none
            - postgresql
            - mongodb
          enumNames:
            - None
            - PostgreSQL
            - MongoDB
        port:
          title: Port
          type: number
          description: Service port
          default: 8080
    - title: Repository
      required:
        - repoUrl
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com
  steps:
    - id: fetch
      name: Fetch Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          serviceName: ${{ parameters.serviceName }}
          team: ${{ parameters.team }}
          description: ${{ parameters.description }}
          database: ${{ parameters.database }}
          port: ${{ parameters.port }}
          year: ${{ parameters.year }} # Collected as a scaffolder parameter
          templateVersion: "1.0.0"
    - id: fetch-database
      name: Add Database Configuration
      if: ${{ parameters.database !== 'none' }}
      action: fetch:template
      input:
        url: ./skeleton/database/${{ parameters.database }}
        targetPath: src/database
        values:
          serviceName: ${{ parameters.serviceName }}
          database: ${{ parameters.database }}
    - id: fetch-infra-claim
      name: Add Infrastructure Claim
      if: ${{ parameters.database !== 'none' }}
      action: fetch:template
      input:
        url: ./skeleton/infrastructure
        targetPath: infrastructure
        values:
          serviceName: ${{ parameters.serviceName }}
          team: ${{ parameters.team }}
          database: ${{ parameters.database }}
    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        allowedHosts: ["github.com"]
        repoUrl: ${{ parameters.repoUrl }}
        description: ${{ parameters.description }}
        defaultBranch: main
        protectDefaultBranch: true
        requireCodeOwnerReviews: true
    - id: create-namespace
      name: Create Kubernetes Namespace
      action: http:backstage:request
      input:
        method: POST
        path: /api/onboarding/v1/namespaces
        headers:
          Content-Type: application/json
        body:
          name: ${{ parameters.serviceName }}
          team: ${{ parameters.team }}
          labels:
            platform.io/created-by: backstage-scaffolder
    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: Open in Catalog
        icon: catalog
        entityRef: ${{ steps.register.output.entityRef }}
    text:
      - title: Next Steps
        content: |
          Your service has been created. Clone the repository and start development:
          ```bash
          git clone ${{ steps.publish.output.remoteUrl }}
          cd ${{ parameters.serviceName }}
          docker compose up
          ```
```

---

### Archivos de la Plantilla (*Template Files*)

#### 1. Archivo `package.json` con Metadatos de Plataforma

```json
{
  "name": "${{ values.serviceName }}",
  "version": "0.1.0",
  "description": "${{ values.description }}",
  "main": "dist/index.js",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write src/",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "express": "^5.0.0",
    "pino": "^8.16.0",
    "pino-http": "^8.5.0",
    "@opentelemetry/api": "^1.6.0",
    "@opentelemetry/sdk-node": "^0.44.0",
    "@opentelemetry/auto-instrumentations-node": "^0.39.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.20",
    "@types/node": "^20.9.0",
    "typescript": "^5.2.2",
    "tsx": "^4.1.1",
    "vitest": "^2.0.0",
    "eslint": "^9.0.0",
    "prettier": "^3.1.0"
  },
  "engines": {
    "node": ">=20.0.0"
  },
  "platformMetadata": {
    "team": "${{ values.team }}",
    "templateName": "backend-service",
    "templateVersion": "${{ values.templateVersion }}",
    "generatedAt": "${{ values.year }}"
  }
}
```

#### 2. `Dockerfile` Estandarizado Multi-Etapa

```dockerfile
# templates/backend-service/v1/skeleton/Dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS runner
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 appuser
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
RUN chown -R appuser:nodejs /app
USER appuser
ENV NODE_ENV=production
ENV PORT=${{ values.port }}
EXPOSE ${{ values.port }}
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:${{ values.port }}/health', \
  (r) => { process.exit(r.statusCode === 200 ? 0 : 1) })"
CMD ["node", "dist/index.js"]
```

#### 3. Flujo de CI/CD Integrado con Acciones de Plataforma

```yaml
# templates/backend-service/v1/skeleton/.github/workflows/ci.yml
name: CI/CD Pipeline
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  pipeline:
    uses: platform-org/platform-actions/.github/workflows/backend-pipeline.yml@v1
    with:
      service-name: ${{ values.serviceName }}
      node-version: "20"
      deploy-environment: ${{ github.ref == 'refs/heads/main' && 'staging' || 'preview' }}
    secrets:
      registry-token: ${{ secrets.GHCR_TOKEN }}
      kubeconfig: ${{ secrets.KUBECONFIG }}
```

---

### El Problema de las Actualizaciones (*The Upgrade Problem*)

Una vez generado un proyecto, este se desacopla de la plantilla original. Para mitigar la divergencia sin sobreescribir personalizaciones legítimas, se utiliza un paquete de manifiesto de plantilla combinado con **Renovate** [4]:

> **Figura 10.2** - Uso de Renovate para automatizar el seguimiento y notificación de actualizaciones en Starter Kits

```json
// En el package.json del proyecto generado:
{
  "devDependencies": {
    "@platform/template-manifest": "^1.0.0"
  }
}

// @platform/template-manifest/package.json (publicado en el registro interno):
{
  "name": "@platform/template-manifest",
  "version": "1.2.0",
  "description": "Template version tracking for platform starter kits",
  "templateVersions": {
    "backend-service": {
      "latest": "1.2.0",
      "changelog": "https://github.com/platform-org/starter-kits/blob/main/templates/backend-service/CHANGELOG.md"
    }
  }
}
```

Renovate detecta la nueva versión del manifiesto y genera automáticamente una *Pull Request* con enlaces a la guía de migración correspondiente.

---

### Configuración para Desarrollo Local

Configuración de `docker-compose.yml` para reproducir las dependencias de la plataforma (bases de datos y recolector OpenTelemetry) en la máquina local:

```yaml
# templates/backend-service/v1/skeleton/docker-compose.yml
services:
  app:
    build:
      context: .
      target: builder
    ports:
      - "${{ values.port }}:${{ values.port }}"
    volumes:
      - ./src:/app/src
      - ./tests:/app/tests
    environment:
      - NODE_ENV=development
      - LOG_LEVEL=debug
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      {%- if values.database == 'postgresql' %}
      - DATABASE_URL=postgresql://postgres:postgres@postgres:5432/${{ values.serviceName }}
      {%- endif %}
      {%- if values.database == 'mongodb' %}
      - MONGODB_URL=mongodb://mongo:27017/${{ values.serviceName }}
      {%- endif %}
    command: npm run dev
    depends_on:
      {%- if values.database == 'postgresql' %}
      postgres:
        condition: service_healthy
      {%- endif %}
      {%- if values.database == 'mongodb' %}
      mongo:
        condition: service_healthy
      {%- endif %}

  {%- if values.database == 'postgresql' %}
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${{ values.serviceName }}
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
  {%- endif %}

  {%- if values.database == 'mongodb' %}
  mongo:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh localhost:27017/test --quiet
      interval: 5s
      timeout: 5s
      retries: 5
  {%- endif %}

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    ports:
      - "4317:4317"
      - "4318:4318"
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
    command: ["--config=/etc/otelcol-contrib/config.yaml"]

volumes:
  {%- if values.database == 'postgresql' %}
  postgres-data:
  {%- endif %}
  {%- if values.database == 'mongodb' %}
  mongo-data:
  {%- endif %}
```

#### Script de Ayuda para el Flujo de Desarrollo (`dev.py`)

```python
# templates/backend-service/v1/skeleton/dev.py
#!/usr/bin/env python3
"""Development helper scripts for starter kit projects."""
import subprocess
import sys
from pathlib import Path
from typing import Optional

def run_command(cmd: list[str], cwd: Optional[Path] = None) -> int:
    """Run a command and return exit code."""
    result = subprocess.run(cmd, cwd=cwd)
    return result.returncode

def dev_start():
    """Start local development environment."""
    print("Starting development environment...")
    compose_file = Path("docker-compose.yml")
    if compose_file.exists():
        return run_command(["docker", "compose", "up", "--build"])
    else:
        return run_command(["npm", "run", "dev"])

def dev_test(watch: bool = False):
    """Run tests."""
    cmd = ["npm", "run", "test:watch"] if watch else ["npm", "test"]
    return run_command(cmd)

def dev_lint(fix: bool = False):
    """Run linting."""
    cmd = ["npm", "run", "lint:fix"] if fix else ["npm", "run", "lint"]
    return run_command(cmd)

def dev_clean():
    """Clean build artifacts and dependencies."""
    print("Cleaning project...")
    dirs_to_remove = ["node_modules", "dist", ".turbo", "coverage"]
    for dir_name in dirs_to_remove:
        dir_path = Path(dir_name)
        if dir_path.exists():
            print(f"  Removing {dir_name}/")
            import shutil
            shutil.rmtree(dir_path)
    if Path("docker-compose.yml").exists():
        run_command(["docker", "compose", "down", "-v"])
    print("Clean complete!")
    return 0

def dev_validate():
    """Validate project configuration."""
    print("Validating project configuration...")
    checks = [
        ("package.json exists", Path("package.json").exists()),
        ("Dockerfile exists", Path("Dockerfile").exists()),
        ("CI workflow exists", Path(".github/workflows/ci.yml").exists()),
        ("README exists", Path("README.md").exists()),
    ]
    all_passed = True
    for name, passed in checks:
        status = "PASS" if passed else "FAIL"
        print(f"  [{status}] {name}")
        if not passed:
            all_passed = False
    return 0 if all_passed else 1

if __name__ == "__main__":
    commands = {
        "start": dev_start,
        "test": lambda: dev_test(watch="--watch" in sys.argv),
        "lint": lambda: dev_lint(fix="--fix" in sys.argv),
        "clean": dev_clean,
        "validate": dev_validate,
    }
    if len(sys.argv) < 2 or sys.argv[1] not in commands:
        print("Usage: dev.py <command>")
        print("Commands:", ", ".join(commands.keys()))
        sys.exit(1)
    sys.exit(commands[sys.argv[1]]())
```

---

### Patrones de Fallo y Salvaguardas

- **Publicación de plantillas defectuosas**: Mitigado mediante pruebas automatizadas en CI que generan un proyecto completo y ejecutan compilación, linters y tests antes de permitir el registro en el catálogo.
- **Personalización excesiva**: Delimitar claramente qué archivos pertenecen a la plataforma (`Dockerfile`, flujos de CI, configuraciones de seguridad) frente a los archivos del equipo (rutas de API, lógica de negocio).
- **Caídas del portal de desarrolladores**: Mantener un procedimiento alternativo de emergencia clonando el repositorio de plantillas directamente.
- **Diferencias entre entorno local y CI**: Probar las plantillas en entornos y contenedores equivalentes a los ejecutores de CI.

---

### Publicación y Pruebas Automatizadas de Plantillas

El script `publish.py` automatiza el descubrimiento, validación sintáctica y registro de ubicaciones en la API del catálogo de Backstage (`POST /api/catalog/locations`).

La suite de pruebas (`test_templates.py`) valida las plantillas en dos niveles:
1. **Pruebas Estructurales**: Verifican que existan los archivos obligatorios del esqueleto (`Dockerfile`, `README.md`, `ci.yml`).
2. **Pruebas de Comportamiento**: Generan un proyecto de prueba interpolando variables, ejecutan `npm install`, `npm run build`, `npm run lint`, `npm test` y levantan el proceso para comprobar que responde al *endpoint* de salud `/health`.

---

### Creación de un Nuevo Servicio y Validación Integral

Script de validación del flujo completo de creación de servicios (`validate-workflow.py`):

```python
# validate-workflow.py
#!/usr/bin/env python3
"""Validate the complete starter kit workflow."""
import subprocess
import time
import sys
from pathlib import Path

def run(cmd: list[str], cwd: Path = None) -> tuple[int, str]:
    """Run command and return exit code and output."""
    result = subprocess.run(cmd, cwd=cwd, capture_output=True, text=True)
    return result.returncode, result.stdout + result.stderr

def validate_clone():
    """Validate repository was created and can be cloned."""
    print("Testing repository clone...")
    code, output = run([
        "git", "clone", "https://github.com/platform-org/order-service.git",
        "order-service"
    ])
    if code != 0:
        print(f"Clone failed: {output}")
        return False
    print("Clone: PASSED")
    return True

def validate_local_dev():
    """Validate local development workflow."""
    print("Testing local development...")
    project_path = Path("order-service")
    
    # Install dependencies
    code, _ = run(["npm", "install"], cwd=project_path)
    if code != 0:
        print("npm install failed")
        return False
    
    # Build project
    code, output = run(["npm", "run", "build"], cwd=project_path)
    if code != 0:
        print(f"Build failed: {output}")
        return False
    
    # Run tests
    code, output = run(["npm", "test"], cwd=project_path)
    if code != 0:
        print(f"Tests failed: {output}")
        return False
    
    # Run lint
    code, output = run(["npm", "run", "lint"], cwd=project_path)
    if code != 0:
        print(f"Lint failed: {output}")
        return False
    
    print("Local development: PASSED")
    return True

def validate_container_build():
    """Validate container builds successfully."""
    print("Testing container build...")
    project_path = Path("order-service")
    code, output = run([
        "docker", "build", "-t", "order-service:local", "."
    ], cwd=project_path)
    if code != 0:
        print(f"Container build failed: {output}")
        return False
    print("Container build: PASSED")
    return True

def validate_catalog_registration():
    """Validate service appears in Backstage catalog."""
    print("Testing catalog registration...")
    project_path = Path("order-service")
    catalog_file = project_path / "catalog-info.yaml"
    import yaml
    with open(catalog_file) as f:
        catalog = yaml.safe_load(f)
    if catalog.get("kind") != "Component":
        print("Invalid catalog kind")
        return False
    if not catalog.get("metadata", {}).get("name"):
        print("Missing component name")
        return False
    print("Catalog registration: PASSED")
    return True

def validate_infrastructure_claim():
    """Validate database infrastructure claim exists."""
    print("Testing infrastructure claim...")
    project_path = Path("order-service")
    claim_file = project_path / "infrastructure" / "database.yaml"
    if not claim_file.exists():
        print("Database claim file missing")
        return False
    import yaml
    with open(claim_file) as f:
        claim = yaml.safe_load(f)
    if claim.get("kind") != "PostgreSQLClaim":
        print("Invalid claim kind")
        return False
    print("Infrastructure claim: PASSED")
    return True

def cleanup():
    """Clean up test artifacts."""
    import shutil
    project_path = Path("order-service")
    if project_path.exists():
        shutil.rmtree(project_path)

def main():
    cleanup()
    tests = [
        validate_clone,
        validate_local_dev,
        validate_container_build,
        validate_catalog_registration,
        validate_infrastructure_claim
    ]
    all_passed = True
    for test in tests:
        if not test():
            all_passed = False
            break
    cleanup()
    if all_passed:
        print("\nAll workflow validations PASSED!")
        return 0
    else:
        print("\nWorkflow validation FAILED!")
        return 1

if __name__ == "__main__":
    sys.exit(main())
```

---

### Ejercicio 10.1: Creación y Publicación de una Plantilla de Starter Kit

1. Configurar la estructura de carpetas en el repositorio `starter-kits`.
2. Crear los archivos base del esqueleto (`Dockerfile`, `package.json`, `ci.yml`).
3. Definir la plantilla de Backstage (`template.yaml`) con parámetros y pasos condicionales.
4. Incluir el bloque `platformMetadata` en `package.json` para el seguimiento de versiones.
5. Ejecutar la suite de pruebas automatizadas para validar compilación, linters y tests.
6. Publicar la plantilla en el catálogo de Backstage mediante el script de publicación.
7. Crear un nuevo servicio desde la interfaz web del portal.
8. Clonar el repositorio generado, probar el entorno local con Docker Compose y comprobar el registro en el catálogo.

---

### Resumen

- **Estandarización sin fricción**: Los *starter kits* codifican las mejores prácticas de la organización, permitiendo crear proyectos listos para producción en minutos.
- **Desacoplamiento arquitectónico**: Separación clara entre archivos de plantilla, lógica de andamiaje (*scaffolding*) y mecanismos de distribución.
- **Integración con Backstage Scaffolder**: Aprovecha la identidad y el catálogo corporativo para registrar componentes de forma guiada.
- **Estrategia de actualización con Renovate**: Notifica a los equipos sobre nuevas versiones de plantillas mediante paquetes de manifiesto sin sobreescribir código personalizado.
- **Pruebas integrales de comportamiento**: Garantizan que las plantillas generen proyectos funcionales y sin errores antes de su publicación en el catálogo.

---

### Referencias

- **[1]** *Yeoman Authoring Documentation*. [https://yeoman.io/authoring/](https://yeoman.io/authoring/)
- **[2]** *Backstage Software Templates Overview*. [https://backstage.io/docs/features/software-templates/](https://backstage.io/docs/features/software-templates/)
- **[3]** *Writing Backstage Templates*. [https://backstage.io/docs/features/software-templates/writing-templates](https://backstage.io/docs/features/software-templates/writing-templates)
- **[4]** *Renovate Documentation*. [https://docs.renovatebot.com/](https://docs.renovatebot.com/)
