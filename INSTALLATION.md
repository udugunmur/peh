# Apéndice A: Configuración del conjunto de herramientas organizacionales

**The Platform Engineer's Handbook**
*Guía completa de configuración para macOS, Linux y Windows*

---

Antes de comenzar a programar, deberás asegurarte de que las herramientas utilizadas estén configuradas para una organización simulada que nos permita practicar técnicas de ingeniería de plataformas a escala. Para obtener detalles sobre cómo realizar algunas de estas operaciones, incluiremos enlaces a la documentación de los proveedores y de las herramientas.

## Tabla de contenidos

- [Configuración de cuentas](#configuracion-de-cuentas)
  - [Configuración de cuenta de Pulumi](#configuracion-de-cuenta-de-pulumi)
  - [Configuración de cuenta de GitHub](#configuracion-de-cuenta-de-github)
  - [Configuración de CircleCI](#configuracion-de-circleci)
  - [Configuración de Bitwarden](#configuracion-de-bitwarden)
- [Herramientas fundamentales](#herramientas-fundamentales)
  - [Gestores de paquetes](#gestores-de-paquetes)
  - [Git](#git)
  - [Docker](#docker)
  - [Python 3.10+ y UV](#python-310-y-uv)
  - [Node.js 18+ y npm](#nodejs-18-y-npm)
  - [kubectl](#kubectl)
  - [Kind (Kubernetes in Docker)](#kind-kubernetes-in-docker)
  - [Helm](#helm)
- [Instrucciones de instalación específicas por capítulo](#instrucciones-de-instalacion-especificas-por-capitulo)
- [Solución de problemas](#solucion-de-problemas)
- [Referencia rápida: Herramientas por capítulo](#referencia-rapida-herramientas-por-capitulo)

---

## Configuración de cuentas

### Configuración de cuenta de Pulumi

1. Regístrate para obtener una cuenta gratuita de Pulumi en [pulumi.com](https://pulumi.com).
2. Ten en cuenta que para los ejercicios de este libro no se requiere una cuenta de Organización; una cuenta Individual es suficiente.
3. Usando el enlace de perfil en la esquina superior derecha de la página, crea un **Personal Access Token** (Token de Acceso Personal).
4. Toma nota de este valor por ahora; lo guardaremos en un almacén de secretos más adelante.

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Olvidar aplicar un backend de estado coherente (local vs. nube) provoca desviaciones (*drift*).
> - Crear múltiples tokens sin registrarlos debidamente dificulta su revocación.
> - Omitir salvaguardas de políticas como código permite la entrada de patrones de infraestructura inseguros.

### Configuración de cuenta de GitHub

Recomendamos crear una organización de GitHub dedicada para practicar con este libro. Aunque puedes usar un repositorio personal, no te dará acceso a algunas de las opciones organizacionales empleadas, ni a la capacidad de añadir cuentas de desarrollo "simuladas" para diferentes roles.

1. Crea una nueva Organización siguiendo la documentación en [GitHub Docs: Crear una nueva organización](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch). Usa un nombre apropiado como `<<tunombre>>-peh-org`.
2. Crea un Personal Access Token de grano fino (Fine-Grained PAT) autorizado para su uso por la organización siguiendo la [documentación de GitHub PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens).
3. Se requieren los siguientes permisos para que el PAT funcione con los ejercicios de este libro:
   - Acceso a todos los repositorios de tu organización (no a tu cuenta personal)
   - Administration: Read & Write
   - Commit statuses: Read & Write
   - Contents: Read & Write
   - Custom Properties: Read & Write
   - Metadata: Read-Only
4. Toma nota de este token por ahora; lo guardaremos en un almacén de secretos más adelante.

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - No habilitar SSO y 2FA crea una postura de seguridad débil y arriesga la pérdida de acceso a tus archivos.
> - Omitir las protecciones de ramas puede provocar fusiones accidentales en la rama principal (`main`).
> - Mezclar repositorios personales con los de la organización fragmenta la propiedad y el acceso.

### Configuración de CircleCI

1. Crea una cuenta gratuita de CircleCI utilizando una dirección de correo electrónico en [circleci.com](https://circleci.com).
2. En la página principal, crea una nueva Organización (puedes usar el mismo nombre que tu Organización de GitHub).
3. En la configuración de la Organización (Organization Settings), configura una conexión VCS hacia todos los repositorios de tu cuenta de Organización de GitHub.
4. En Organization Settings → Self-hosted runners, acepta los términos y condiciones para utilizar un ejecutor local (*local runner*). *Configuraremos el runner en sí más adelante utilizando Kind.*

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Confiar únicamente en la interfaz web para configurar pipelines en lugar de YAML bajo control de versiones causará problemas de reproducibilidad.
> - El uso excesivo del nivel gratuito sin optimizar trabajos provocará alcanzar los límites con rapidez.
> - Los ejecutores autoalojados mal configurados consumen recursos locales y generan compilaciones inestables.

### Configuración de Bitwarden

1. Crea una cuenta gratuita de Bitwarden en [bitwarden.com](https://bitwarden.com) utilizando una dirección de correo electrónico.
2. Guarda de forma segura tu contraseña maestra de Bitwarden. Será necesaria más adelante, y perderás acceso a tu bóveda sin ella.
3. Crea una clave de API siguiendo la documentación en [Bitwarden API Key docs](https://bitwarden.com/help/personal-api-key/).
4. Toma nota del Client ID y Client Secret por ahora; crearemos scripts utilizándolos como variables de entorno más adelante.

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Los equipos continúan almacenando secretos en configuraciones en lugar de migrarlos a la bóveda.
> - Una delimitación deficiente del acceso (todos los usuarios ven todos los secretos) genera problemas de cumplimiento normativo.
> - Los secretos extraídos incorrectamente en los pipelines acaban apareciendo expuestos en los registros (*logs*).

---

## Herramientas fundamentales

Estas herramientas se utilizan a lo largo de múltiples capítulos y deben instalarse primero. Se proporcionan instrucciones para macOS, Linux (Ubuntu/Debian) y Windows. Siempre que sea posible, recomendamos utilizar gestores de paquetes (Homebrew para macOS, apt para Linux y Chocolatey/winget para Windows) para simplificar la instalación y las actualizaciones.

### Gestores de paquetes

Los gestores de paquetes simplifican la instalación y actualización de software. Configura el correspondiente a tu sistema operativo antes de instalar otras herramientas.

#### Homebrew (macOS)

Homebrew es el gestor de paquetes recomendado para macOS.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### Chocolatey (Windows)

Abre PowerShell como Administrador y ejecuta:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString("https://community.chocolatey.org/install.ps1"))
```

#### apt (Linux)

apt viene preinstalado en Ubuntu/Debian. Mantenlo actualizado:

```bash
sudo apt update && sudo apt upgrade -y
```

### Git

Sistema de control de versiones distribuido utilizado en todos los capítulos para la gestión de código fuente y colaboración.

**macOS:**
```bash
brew install git
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install -y git
```

**Windows:**
```powershell
choco install git -y
# o bien: winget install Git.Git
```

**Verificar instalación:**
```bash
git --version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - No configurar `user.name` y `user.email` globalmente produce confirmaciones (*commits*) anónimas o mal atribuidas.
> - Ignorar las mejores prácticas de `.gitignore` hace que secretos, artefactos de compilación o configuraciones de IDE se filtren en los repositorios.
> - Trabajar directamente en `main` sin ramas de características (*feature branches*) dificulta la colaboración y hace traumáticas las reversiones (*rollbacks*).

### Docker

Entorno de ejecución de contenedores requerido para compilar imágenes y ejecutar clústeres de Kind. Docker Desktop proporciona tanto el demonio como la CLI.

**macOS:**
```bash
brew install --cask docker
# A continuación inicia Docker Desktop desde Aplicaciones
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER  # Cierra sesión y vuelve a iniciarla tras esto
```

**Windows:**
```powershell
choco install docker-desktop -y
# A continuación inicia Docker Desktop y habilita el backend de WSL 2
```

**Verificar instalación:**
```bash
docker --version
docker compose version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Ejecutar Docker Desktop con la memoria predeterminada (2 GB) es insuficiente para clústeres de Kind; aumenta al menos a 4 GB.
> - Olvidar limpiar imágenes y volúmenes no utilizados llena rápidamente el disco (usa `docker system prune` con regularidad).
> - En Linux, saltarse el paso `usermod -aG docker` obliga a que cada comando de Docker requiera `sudo`.

### Python 3.10+ y UV

Python se utiliza para herramientas de plataforma, scripts de automatización y servicios agénticos de IA. UV es un instalador y gestor de paquetes de Python extremadamente rápido.

**macOS:**
```bash
brew install python@3.11
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install -y python3 python3-pip python3-venv
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Windows:**
```powershell
choco install python --version=3.11.0 -y
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

**Verificar instalación:**
```bash
python3 --version  # o 'python --version' en Windows
uv --version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Instalar paquetes globalmente en lugar de usar entornos virtuales provoca conflictos de dependencias en todo el sistema.
> - La confusión entre `python` y `python3` en sistemas basados en Unix puede apuntar a una versión de Python obsoleta del sistema.
> - Olvidar añadir `~/.cargo/bin` a tu PATH tras instalar UV hace que el comando `uv` no esté disponible.

### Node.js 18+ y npm

Requerido para Backstage (Capítulo 6), CDK for Kubernetes (cdk8s) y herramientas para desarrolladores frontend.

**macOS:**
```bash
brew install node@18
```

**Linux (Ubuntu/Debian):**
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```

**Windows:**
```powershell
choco install nodejs-lts --version=18.19.0 -y
```

**Verificar instalación:**
```bash
node --version
npm --version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Usar Node.js 20+ con versiones antiguas de Backstage puede causar errores de compatibilidad en la compilación; mantente en Node.js 18 LTS si surgen problemas.
> - No limpiar `node_modules` tras cambiar de versión de Node puede provocar fallos sutiles en tiempo de ejecución.
> - En Linux, omitir la configuración del repositorio NodeSource suele instalar una versión muy anticuada del repositorio base de la distribución.

### kubectl

Herramienta de línea de comandos de Kubernetes para comunicarse con el plano de control del clúster.

**macOS:**
```bash
brew install kubectl
```

**Linux (Ubuntu/Debian):**
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubectl
```

**Windows:**
```powershell
choco install kubernetes-cli -y
# o bien: winget install Kubernetes.kubectl
```

**Verificar instalación:**
```bash
kubectl version --client
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Mantener múltiples contextos de kubeconfig sin herramientas como `kubectx` incrementa el riesgo de aplicar cambios en el clúster erróneo.
> - El sesgo de versión de kubectl superior a una versión menor (+/- 1) respecto a la versión de la API de Kubernetes puede producir advertencias de obsolescencia o llamadas fallidas.
> - Olvidar configurar el autocompletado de bash/zsh ralentiza significativamente la productividad en la terminal.

### Kind (Kubernetes in Docker)

Ejecuta clústeres locales de Kubernetes utilizando contenedores Docker como nodos. Esencial para pruebas locales de plataformas.

**macOS:**
```bash
brew install kind
```

**Linux (Ubuntu/Debian):**
```bash
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

**Windows:**
```powershell
choco install kind -y
# o bien: winget install Kubernetes.kind
```

**Verificar instalación:**
```bash
kind version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - No mapear puertos de host adicionales en la configuración de Kind impide acceder a NodePort o servicios de ingress desde tu máquina local.
> - Crear demasiados clústeres simultáneos de Kind satura la memoria de Docker y deja sin respuesta al daemon.
> - Olvidar exportar o fusionar los contextos kubeconfig hace que `kubectl` no encuentre el clúster recién creado.

### Helm

Gestor de paquetes para Kubernetes utilizado para desplegar operadores, pilas de observabilidad y servicios de plataforma.

**macOS:**
```bash
brew install helm
```

**Linux (Ubuntu/Debian):**
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update && sudo apt install -y helm
```

**Windows:**
```powershell
choco install kubernetes-helm -y
# o bien: winget install Helm.Helm
```

**Verificar instalación:**
```bash
helm version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Olvidar ejecutar `helm repo update` antes de instalar charts suele causar fallos de resolución de versiones o dependencias ausentes.
> - Instalar charts sin fijar `--version` provoca actualizaciones no deseadas en futuros despliegues automáticos.
> - Sobrescribir demasiados valores por defecto sin comprender la estructura del chart conduce a despliegues mal configurados.

---
## Instrucciones de instalación específicas por capítulo

Las siguientes secciones detallan las herramientas adicionales requeridas para cada capítulo más allá de las herramientas fundamentales indicadas anteriormente.

### Capítulo 1: Sentando las bases

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Pulumi | ≥3.0 | Motor de Infraestructura como Código (IaC) |
| Bitwarden CLI | ≥2024.1 | Gestión de secretos |
| CircleCI CLI | Latest | Gestión de pipelines de CI/CD |
| pre-commit | Latest | Framework de hooks de Git para calidad de código |
| pytest | ≥7.0 | Pruebas unitarias en Python |

#### Pulumi

Herramienta de Infraestructura como Código que utiliza Python. La configuración de CircleCI en este capítulo demuestra flujos de trabajo de previsualización y aplicación (`preview`/`apply`) con Pulumi. Crea una cuenta gratuita en [pulumi.com](https://pulumi.com) antes de instalar la CLI.

**macOS:**
```bash
brew install pulumi/tap/pulumi
```

**Linux (Ubuntu/Debian):**
```bash
curl -fsSL https://get.pulumi.com | sh
echo "export PATH=$HOME/.pulumi/bin:$PATH" >> ~/.bashrc && source ~/.bashrc
```

**Windows:**
```powershell
choco install pulumi -y
```

**Verificar instalación:**
```bash
pulumi version
pulumi login  # Autentícate con tu cuenta de Pulumi
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Olvidar ejecutar `pulumi login` antes de `pulumi up` hace que la CLI solicite credenciales interactivamente, interrumpiendo la automatización.
> - No cifrar secretos con `pulumi config set --secret` expone valores confidenciales en texto plano en el estado del stack.
> - Dejar stacks huérfanos en la consola de Pulumi tras eliminar recursos dificulta saber qué está realmente desplegado.

#### Bitwarden CLI

Interfaz de línea de comandos para el gestor de contraseñas Bitwarden. Crea una cuenta gratuita en [bitwarden.com](https://bitwarden.com) y genera una clave de API antes de utilizarla.

**macOS:**
```bash
brew install bitwarden-cli
```

**Linux (Ubuntu/Debian):**
```bash
sudo snap install bw
```

**Windows:**
```powershell
choco install bitwarden-cli -y
```

**Verificar instalación:**
```bash
bw --version
```

#### CircleCI CLI

Herramienta de línea de comandos para validar configuraciones de CircleCI y gestionar pipelines. Crea una cuenta gratuita en [circleci.com](https://circleci.com).

**macOS:**
```bash
brew install circleci
```

**Linux (Ubuntu/Debian):**
```bash
curl -fLSs https://raw.githubusercontent.com/CircleCI-CLI/circleci-cli/main/install.sh | bash
```

**Windows:**
```powershell
# Descarga desde https://github.com/CircleCI-CLI/circleci-cli/releases
# Extrae y añade a tu PATH
```

**Verificar instalación:**
```bash
circleci version
circleci setup  # Configura con tu token de API
```

#### pre-commit

Framework para gestionar hooks de Git multi-lenguaje previos a la confirmación (*pre-commit*). Se utiliza para imponer convenciones en los mensajes de confirmación, ejecutar linters y validar configuraciones antes de confirmar el código.

**macOS:**
```bash
brew install pre-commit
```

**Linux (Ubuntu/Debian):**
```bash
pip3 install pre-commit --break-system-packages
```

**Windows:**
```powershell
pip3 install pre-commit
```

**Verificar instalación:**
```bash
pre-commit --version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - No ejecutar `pre-commit install` tras clonar un repositorio hace que los hooks nunca se activen para ese desarrollador.
> - Hooks excesivamente estrictos que tardan demasiado tiempo en ejecutarse incitan a los desarrolladores a omitirlos con `--no-verify`.
> - No fijar las versiones de los hooks en `.pre-commit-config.yaml` provoca comportamientos inconsistentes entre los miembros del equipo.

#### Herramientas de prueba y calidad en Python

Instala las herramientas de pruebas de Python en un entorno virtual o de forma global:

```bash
pip3 install pytest pytest-cov pyyaml
# En Linux, añade --break-system-packages si instalas globalmente
```

---

### Capítulo 2: Construcción de un entorno de ejecución de plataforma escalable

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Flux CD | ≥2.0 | Entrega continua orientada a GitOps |
| Istio | ≥1.20 | Malla de servicios (*service mesh*) con mTLS |
| Kustomize | ≥5.0 | Gestión de configuraciones de Kubernetes |
| bats-core | ≥1.10 | Sistema automatizado de pruebas en Bash (Bash Automated Testing System) |

#### Flux CD

Kit de herramientas de GitOps para Kubernetes. Flux reconcilia continuamente el estado del clúster con los repositorios de Git.

**macOS:**
```bash
brew install fluxcd/tap/flux
```

**Linux (Ubuntu/Debian):**
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

**Windows:**
```powershell
choco install flux -y
# O descarga desde https://github.com/fluxcd/flux2/releases
```

**Verificar instalación:**
```bash
flux --version
flux check --pre  # Comprobar prerrequisitos
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - No inicializar (*bootstrap*) Flux con los permisos correctos en el token de GitHub produce fallos silenciosos de sincronización.
> - Modificar recursos directamente con `kubectl` en lugar de a través de Git rompe el bucle de reconciliación de GitOps.
> - Olvidar configurar notificaciones en Flux implica que las desviaciones de configuración pasen inadvertidas durante horas o días.

#### Istio (istioctl)

Malla de servicios que proporciona cifrado mTLS, gestión de tráfico y observabilidad. Se instala mediante `istioctl`.

**macOS:**
```bash
brew install istioctl
```

**Linux (Ubuntu/Debian):**
```bash
curl -L https://istio.io/downloadIstio | sh -
sudo mv istio-*/bin/istioctl /usr/local/bin/
```

**Windows:**
```powershell
# Descarga desde https://github.com/istio/istio/releases
# Extrae y añade istioctl.exe a tu PATH
```

**Verificar instalación:**
```bash
istioctl version
istioctl install --set profile=demo -y  # Instalar en el clúster
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Instalar Istio sin límites de recursos puede consumir memoria significativa del clúster (1-2 GB para el plano de control).
> - Olvidar etiquetar los espacios de nombres (*namespaces*) con `istio-injection=enabled` provoca que los sidecars no se inyecten en los pods.
> - Actualizar Istio sin seguir el proceso de actualización canary puede causar breves interrupciones del tráfico.

#### Kustomize

Personalización sin plantillas de configuraciones YAML de Kubernetes.

**macOS:**
```bash
brew install kustomize
```

**Linux (Ubuntu/Debian):**
```bash
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

**Windows:**
```powershell
choco install kustomize -y
```

**Verificar instalación:**
```bash
kustomize version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Utilizar la versión integrada de kustomize en kubectl (`kubectl apply -k`) puede quedar rezagada respecto a la versión independiente y carecer de funciones más recientes.
> - Superposiciones (*overlays*) profundamente anidadas se vuelven difíciles de depurar; mantén la jerarquía poco profunda (base + una o dos superposiciones).
> - Olvidar incluir nuevos archivos en la lista de recursos de `kustomization.yaml` hace que sean ignorados silenciosamente.

#### bats-core

Framework de pruebas para scripts de Bash, utilizado para pruebas de validación de infraestructura.

**macOS:**
```bash
brew install bats-core
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install -y bats
```

**Windows:**
```powershell
npm install -g bats
```

**Verificar instalación:**
```bash
bats --version
```

---

### Capítulo 3: Asegurando el acceso a la plataforma

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Keycloak | ≥22.0 | Gestión de identidad y acceso (OIDC/OAuth2) |
| OPA Gatekeeper | ≥3.14 | Controlador de admisión de Kubernetes para cumplimiento de políticas |
| cert-manager | ≥1.13 | Gestión automatizada de certificados TLS |

#### Keycloak

Proveedor de identidad de código abierto compatible con OIDC, OAuth2 y SAML. Despliégalo en tu clúster de Kind mediante Helm.

**Todas las plataformas (Helm es multiplataforma):**
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install keycloak bitnami/keycloak \
  --namespace keycloak --create-namespace \
  --set auth.adminUser=admin \
  --set auth.adminPassword=admin
```

**Verificar instalación:**
```bash
kubectl get pods -n keycloak
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Mantener las credenciales predeterminadas `admin`/`admin` en un entorno compartido es un riesgo crítico de seguridad.
> - No configurar almacenamiento persistente provoca que el estado de Keycloak (realms, usuarios) se pierda al reiniciar el pod.
> - Importar configuraciones de realm manualmente mediante la interfaz web en lugar de exportaciones JSON hace que las instalaciones no sean reproducibles.

#### OPA Gatekeeper

Controlador de políticas para Kubernetes que aplica políticas escritas en Rego en el momento de la admisión.

**Todas las plataformas (Helm es multiplataforma):**
```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system --create-namespace \
  --set enableGenerateViolationEvents=true \
  --set constraintViolationsLimit=1000 \
  --set auditIntervalSeconds=60
```

**Verificar instalación:**
```bash
kubectl get pods -n gatekeeper-system
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Desplegar restricciones en modo de aplicación (*enforce mode*) sin probar primero bloquea cargas de trabajo legítimas en el clúster.
> - No utilizar el modo de prueba (*dryrun mode*) durante el despliegue inicial genera caídas cuando las políticas rechazan inesperadamente despliegues válidos.
> - Olvidar fijar `constraintViolationsLimit` con un valor suficientemente alto hace que Gatekeeper deje de registrar infracciones silenciosamente tras alcanzar el límite por defecto.

#### cert-manager

Automatiza la gestión de certificados TLS dentro de clústeres de Kubernetes.

**Todas las plataformas (Helm es multiplataforma):**
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

**Verificar instalación:**
```bash
kubectl get pods -n cert-manager
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Omitir `--set installCRDs=true` deja a cert-manager incapacitado para crear recursos Certificate e Issuer.
> - Usar el emisor de pruebas (staging) de Let's Encrypt sin advertirlo produce certificados no confiables en producción.
> - No monitorizar la caducidad de certificados hace que los servicios fallen silenciosamente cuando los certificados expiran.

---

### Capítulo 4: Integración de la observabilidad

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| OpenTelemetry SDK | Latest | Librerías de instrumentación de telemetría (Python) |
| Prometheus | ≥2.45 | Recolección y almacenamiento de métricas |
| Grafana | ≥10.0 | Visualización de métricas y paneles de control |
| Jaeger | ≥1.50 | Backend de rastreo distribuido (*tracing*, opcional) |
| Loki | ≥2.9 | Sistema de agregación de registros (*logs*, opcional) |

#### Pila de observabilidad (Despliegue con Helm)

La pila de observabilidad se despliega preferentemente en tu clúster de Kind mediante charts de Helm.

**Prometheus y Grafana (kube-prometheus-stack):**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - El chart `kube-prometheus-stack` consume muchos recursos; en clústeres de Kind, deshabilita componentes que no necesites (por ejemplo, alertmanager).
> - La retención predeterminada (15 días) llena el disco rápidamente en clústeres pequeños; ajusta `server.retention` acorde a tu almacenamiento disponible.
> - No crear recursos `ServiceMonitor` impide que Prometheus descubra las métricas de tu aplicación.

**Jaeger (opcional):**
```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm install jaeger jaegertracing/jaeger \
  --namespace observability --create-namespace
```

**Loki (opcional):**
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace observability --set grafana.enabled=false
```

#### Librerías de OpenTelemetry en Python

Instala el SDK y exportadores de OpenTelemetry para Python:

```bash
pip3 install opentelemetry-api opentelemetry-sdk \
  opentelemetry-exporter-otlp \
  opentelemetry-exporter-prometheus \
  prometheus-client flask
# En Linux, añade --break-system-packages si instalas globalmente
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Mezclar versiones del SDK de OpenTelemetry entre paquetes produce errores de importación y pérdida de atributos.
> - No definir `OTEL_RESOURCE_ATTRIBUTES` para `service.name` hace que las trazas aparezcan como "unknown_service" en Jaeger.
> - Olvidar invocar `shutdown()` en `TracerProvider` durante las pruebas descarta spans silenciosamente.

---

### Capítulo 5: Evaluación de la experiencia de usuario

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| ArgoCD | ≥2.9 | Despliegue continuo orientado a GitOps |
| Flask | ≥2.3 | Framework web de Python (aplicación de demostración) |
| Express.js | ≥4.18 | Framework web de Node.js (demostración de instrumentación) |
| OpenTelemetry JS SDK | ≥1.17 | Instrumentación en Node.js |
| Winston | ≥3.11 | Librería de registro (*logging*) para Node.js |

#### ArgoCD

Herramienta declarativa de entrega continua GitOps para Kubernetes. Utilizada por el script `platform-deploy.sh` para despliegues basados en GitOps.

**Instalar ArgoCD en el clúster:**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Instalar la CLI:**

**macOS:**
```bash
brew install argocd
```

**Linux (Ubuntu/Debian):**
```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

**Windows:**
```powershell
choco install argocd-cli -y
```

**Verificar instalación:**
```bash
argocd version
kubectl get pods -n argocd
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - No recuperar la contraseña inicial de administrador (desde `argocd-initial-admin-secret`) impide iniciar sesión tras la instalación.
> - Sincronizar aplicaciones con auto-sync habilitado antes de disponer de políticas puede desplegar cambios no revisados.
> - El seguimiento de recursos por defecto de ArgoCD puede entrar en conflicto con Flux si ambos están instalados en el mismo clúster.

#### Dependencias de la aplicación demo en Python

```bash
pip3 install flask pyyaml
# En Linux, añade --break-system-packages si instalas globalmente
```

#### Dependencias de instrumentación en Node.js

```bash
npm install express winston
npm install @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions
```

---

### Capítulo 6: Acelerando la experiencia del desarrollador con Backstage

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Backstage | ≥1.20 | Framework de portal para desarrolladores de Spotify |
| PostgreSQL | ≥14.0 | Backend de base de datos para Backstage |

#### Backstage

Portal interno para desarrolladores que ofrece un catálogo de software, TechDocs y plantillas de servicios (*scaffolding*).

**Opción 1: Desplegar mediante Helm**
```bash
helm repo add backstage https://backstage.github.io/charts
helm install backstage backstage/backstage \
  --namespace backstage --create-namespace
```

**Opción 2: Crear instancia local de desarrollo**
```bash
npx @backstage/create-app@latest
```

**Verificar instalación:**
```bash
kubectl get pods -n backstage
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - El archivo `app-config.yaml` de Backstage debe configurarse correctamente para tu organización de GitHub; la configuración predeterminada no descubrirá tus repositorios.
> - Ejecutar Backstage sin PostgreSQL (usando SQLite) sirve para desarrollo, pero falla bajo carga y pierde datos al reiniciar.
> - Los problemas de compatibilidad de plugins son habituales tras actualizar Backstage; revisa siempre las notas de la versión antes de actualizar.

---

### Capítulo 7: Incorporación de equipos en modo autoservicio

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Flask | ≥2.3 | Framework web de Python (API de incorporación / onboarding) |
| @kubernetes/client-node | Latest | Cliente de la API de Kubernetes para Node.js |
| @octokit/rest | Latest | Cliente de la API de GitHub para Node.js |

El Capítulo 7 adopta un enfoque dual: la API principal de incorporación es una aplicación Python/Flask (`onboarding-api.py`), mientras que el servicio de aprovisionamiento de equipos (`services/teamService.js`) muestra el equivalente en Node.js.

**Dependencias de la API de Onboarding en Python:**
```bash
pip3 install flask pyyaml kubernetes
# En Linux, añade --break-system-packages si instalas globalmente
```

**Dependencias del servicio de equipos en Node.js:**
```bash
npm install @kubernetes/client-node @octokit/rest
npm install --save-dev jest supertest
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - El cliente de Kubernetes en Python requiere un kubeconfig válido; ejecutarlo dentro de un contenedor exige configurar el acceso in-cluster.
> - Los permisos del PAT de GitHub deben coincidir con las operaciones (creación de repositorios, gestión de equipos); los alcances (*scopes*) insuficientes causan errores 403 silenciosos.
> - No hacer idempotentes las operaciones de incorporación provoca que reintentar una petición fallida genere namespaces o asignaciones RBAC duplicadas.

---
### Capítulo 8: CI/CD como servicio de plataforma

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| GitHub Actions | N/A | Automatización de flujos de trabajo CI/CD (alojado en GitHub) |
| Trivy | ≥0.48 | Escáner de vulnerabilidades para contenedores |

Los flujos de trabajo de GitHub Actions se ejecutan en ejecutores alojados por GitHub y no requieren instalación local. El capítulo demuestra flujos de trabajo reutilizables y acciones compuestas para un CI/CD estandarizado en la plataforma.

#### Trivy

Escáner de seguridad integral para imágenes de contenedores, sistemas de archivos y configuraciones de IaC.

**macOS:**
```bash
brew install trivy
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
  sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt install -y trivy
```

**Windows:**
```powershell
choco install trivy -y
```

**Verificar instalación:**
```bash
trivy --version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - La base de datos de vulnerabilidades de Trivy debe actualizarse con regularidad; las bases de datos desactualizadas pasan por alto CVEs recién divulgados.
> - Escanear únicamente con `--severity CRITICAL` oculta hallazgos importantes de severidad HIGH que aún requieren remediación.
> - Ejecutar Trivy en CI sin almacenar en caché la descarga de la base de datos añade entre 30 y 60 segundos a cada ejecución del pipeline.

**Dependencias de Python:**
```bash
pip3 install pyyaml requests pytest
# En Linux, añade --break-system-packages si instalas globalmente
```

---

### Capítulo 9: Infraestructura en autoservicio con Crossplane

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Crossplane | ≥1.14 | Gestión de infraestructura nativa de Kubernetes |
| Crossplane CLI | ≥1.14 | CLI para compilar y publicar paquetes de Crossplane |

#### Crossplane

Extiende Kubernetes para gestionar recursos de infraestructura externa mediante Definiciones de Recursos Personalizados (XRDs) y Composiciones.

**Instalar Crossplane en el clúster:**
```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace
```

**Instalar la CLI:**

**macOS:**
```bash
brew install crossplane/tap/crossplane
```

**Linux (Ubuntu/Debian):**
```bash
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv crossplane /usr/local/bin/
```

**Windows:**
```powershell
# Instala la CLI desde los Releases de GitHub
```

**Verificar instalación:**
```bash
crossplane --version
kubectl get pods -n crossplane-system
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - No instalar el proveedor de nube correcto (por ejemplo, `provider-aws`) antes de aplicar Composiciones provoca que los recursos queden bloqueados en estado pendiente (*pending*).
> - Guardar credenciales de proveedores de Crossplane como Secretos de Kubernetes planos sin cifrado en reposo supone un riesgo de seguridad.
> - Modificar esquemas XRD tras el despliegue inicial exige una migración cautelosa; los cambios incompatibles invalidan las reclamaciones existentes.
> - Olvidar aplicar PatchSets para etiquetas de gobernanza genera recursos en la nube sin etiquetas de asignación de costes o propiedad.

**Dependencias de Python:**
```bash
pip3 install flask pyyaml kubernetes
# En Linux, añade --break-system-packages si instalas globalmente
```

---

### Capítulo 10: Publicación de Starter Kits

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Yeoman | Latest | Herramienta de andamiaje para generadores de proyectos |
| Renovate | Latest | Gestión automatizada de actualizaciones de dependencias y plantillas |
| Backstage CLI | Latest | Andamiaje de plantillas de Backstage |

#### Yeoman

Sistema genérico de andamiaje (*scaffolding*) para crear plantillas y generadores de proyectos. Se utiliza para la generación de proyectos basada en CLI en la capa de plantillas.

**Todas las plataformas:**
```bash
npm install -g yo
```

**Verificar instalación:**
```bash
yo --version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Los generadores que codifican rutas de archivos de forma rígida fallan en Windows debido a los diferentes separadores de ruta (utiliza `path.join` en su lugar).
> - No admitir el modo no interactivo (`--no-insight`) imposibilita usar generadores en pipelines de CI/CD.
> - No probar generadores con distintas combinaciones de opciones provoca andamiajes rotos en casos extremos.

#### Renovate

Herramienta automatizada de actualización de dependencias que resuelve el problema de "desviación de plantillas" (*template drift*) descrito en la Sección 10.5. Cuando se actualizan las plantillas de los starter kits, Renovate abre automáticamente solicitudes de extracción (*pull requests*) en los proyectos generados para incorporar las últimas mejoras.

**Opción 1: Instalar la GitHub App de Renovate (recomendado)**
Visita https://github.com/apps/renovate e instálala en tus repositorios.

**Opción 2: CLI autoalojada para pruebas**
```bash
npm install -g renovate
```

**Verificar instalación:**
```bash
renovate --version  # Solo necesario para la opción autoalojada
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - No configurar un archivo `renovate.json` en cada repositorio hace que Renovate omita esos repositorios por completo.
> - Fusionar automáticamente (*auto-merge*) todos los PRs de Renovate sin pruebas implementadas puede introducir cambios incompatibles de forma silenciosa.
> - Reglas de paquetes demasiado amplias (por ejemplo, coincidir con todas las dependencias) generan un exceso de PRs que los equipos aprenden a ignorar.

**Dependencias de Python:**
```bash
pip3 install pyyaml requests pytest
# En Linux, añade --break-system-packages si instalas globalmente
```

---

### Capítulo 11: Validación del cumplimiento normativo con políticas como código

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| OPA Gatekeeper | ≥3.14 | Controlador de admisión de Kubernetes |
| conftest | ≥0.41 | Pruebas de políticas para archivos de configuración |
| OPA CLI | ≥0.60 | Open Policy Agent para pruebas de Rego |
| pre-commit | Latest | Framework de hooks de Git para validación shift-left |
| prometheus-client | Latest | Librería Python para exportadores personalizados de Prometheus |

La instalación de OPA Gatekeeper se cubre en el [Capítulo 3](#capítulo-3-asegurando-el-acceso-a-la-plataforma). Las herramientas adicionales para este capítulo son:

#### conftest

Utilidad para validar datos estructurados contra políticas escritas en Rego. Habilita la validación de políticas en fases tempranas (*shift-left*) tanto en desarrollo como en pipelines de CI.

**macOS:**
```bash
brew install conftest
```

**Linux (Ubuntu/Debian):**
```bash
wget https://github.com/open-policy-agent/conftest/releases/latest/download/conftest_Linux_x86_64.tar.gz
tar xzf conftest_Linux_x86_64.tar.gz
sudo mv conftest /usr/local/bin/
```

**Windows:**
```powershell
choco install conftest -y
# O descarga desde https://github.com/open-policy-agent/conftest/releases
```

**Verificar instalación:**
```bash
conftest --version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Ubicar políticas en un directorio erróneo (`conftest` busca en `policy/` de manera predeterminada) produce "0 tests, 0 failures" sin validación real.
> - No utilizar el modo `--strict` en CI hace que las advertencias pasen desapercibidas; solo los fallos bloquean el pipeline.
> - Políticas en Rego que analizan YAML incorrectamente (por ejemplo, omitiendo `input.metadata`) generan falsos positivos en manifiestos válidos.

#### OPA CLI

La herramienta de línea de comandos de Open Policy Agent para redactar y realizar pruebas unitarias sobre políticas en Rego.

**macOS:**
```bash
brew install opa
```

**Linux (Ubuntu/Debian):**
```bash
curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64
chmod +x opa && sudo mv opa /usr/local/bin/
```

**Windows:**
```powershell
choco install opa -y
# O descarga desde https://www.openpolicyagent.org/docs/latest/#running-opa
```

**Verificar instalación:**
```bash
opa version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Escribir políticas en Rego sin pruebas unitarias (`opa test`) implica que los fallos de lógica solo se detecten en producción durante la admisión.
> - No usar `opa fmt` para formatear políticas provoca inconsistencias de estilo que dificultan las revisiones de código.
> - La semántica de denegación por defecto (*default-deny*) de Rego puede sorprender a los recién llegados; comienza siempre con reglas explícitas de permiso y denegación.

**Dependencias de Python:**
```bash
pip3 install prometheus-client kubernetes pyyaml
# En Linux, añade --break-system-packages si instalas globalmente
```

---

### Capítulo 12: Optimización de costes, rendimiento y escalabilidad

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| OpenCost | ≥1.108 | Asignación de costes para Kubernetes de la CNCF (código abierto) |
| Karpenter | ≥0.33 | Autoescalado de nodos nativo de Kubernetes y selección de instancias |
| VPA | Latest | Vertical Pod Autoscaler para ajuste de tamaño de recursos (*rightsizing*) |
| Metrics Server | Latest | Métricas de recursos del clúster para HPA/VPA |

> **Nota:** HPA (Horizontal Pod Autoscaler) está integrado en cualquier clúster estándar de Kubernetes y no requiere instalación independiente. Solo necesitas el Metrics Server para que el HPA disponga de datos sobre los que operar.

#### OpenCost

Proyecto de la CNCF sandbox que proporciona asignación de costes nativa de Kubernetes. Requiere Prometheus (instalado en el Capítulo 4).

**Todas las plataformas (Helm es multiplataforma):**
```bash
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm repo update
helm install opencost opencost/opencost \
  --namespace opencost --create-namespace \
  --set opencost.prometheus.internal.serviceName="monitoring-kube-prometheus-prometheus" \
  --set opencost.prometheus.internal.namespaceName="monitoring" \
  --set opencost.prometheus.internal.port=9090 \
  --set opencost.ui.enabled=true \
  --set opencost.exporter.defaultClusterId="platform-cluster"
```

**Verificar instalación:**
```bash
kubectl get pods -n opencost
# Interfaz web (puerto 9090) — abre http://localhost:9090
kubectl port-forward -n opencost svc/opencost 9090:9090
# API REST (puerto 9003) — sin interfaz web en este puerto
kubectl port-forward -n opencost svc/opencost 9003:9003
curl http://localhost:9003/allocation/compute?window=24h&aggregate=namespace
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - OpenCost requiere que Prometheus esté en ejecución y accesible; endpoints mal configurados de Prometheus generan informes de coste de 0 $.
> - Sin precios personalizados configurados, OpenCost emplea las tarifas bajo demanda predeterminadas de AWS, que pueden diferir de tus costes reales.
> - En clústeres de Kind, OpenCost reporta 0 $ en cómputo porque no existen instancias reales de la nube para tarificar.

#### Karpenter

Autoescalador de nodos nativo de Kubernetes. Nota: Karpenter está concebido principalmente para AWS EKS. Para otros proveedores en la nube, utiliza Cluster Autoscaler.

```bash
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm install karpenter karpenter/karpenter \
  --namespace karpenter --create-namespace \
  --set settings.aws.clusterName=my-cluster
```

**Verificar instalación:**
```bash
kubectl get pods -n karpenter
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Karpenter requiere perfiles de instancia y roles de IAM específicos; la falta de permisos provoca que los nodos no consigan iniciarse.
> - No definir límites de recursos en NodePools puede generar escalados descontrolados y facturas imprevistas en la nube.
> - Karpenter no opera sobre clústeres locales o Kind; los ejercicios de Karpenter requieren un clúster EKS real.

#### Vertical Pod Autoscaler (VPA)

VPA no está incluido en Kubernetes estándar y debe instalarse por separado. Proporciona recomendaciones sobre solicitudes de recursos.

**Todas las plataformas (requiere acceso de kubectl al clúster):**
```bash
git clone https://github.com/kubernetes/autoscaler.git /tmp/autoscaler
kubectl apply -f /tmp/autoscaler/vertical-pod-autoscaler/deploy/
# Ignora los errores de CRDs v1beta1 — son inofensivos (versiones antiguas de la API)
```

**Verificar instalación:**
```bash
kubectl get pods -n kube-system | grep vpa
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - VPA en modo Auto reinicia pods para aplicar nuevas solicitudes de recursos; esto puede interrumpir cargas de trabajo con estado (*stateful*).
> - VPA y HPA no deben tener como objetivo el uso de CPU en el mismo despliegue; entrarán en conflicto mutuo.
> - Las recomendaciones de VPA tardan tiempo en estabilizarse (entre 24 y 48 horas de datos); no tomes decisiones con las recomendaciones iniciales.

#### Metrics Server

Requerido para la operatividad de HPA y VPA:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# Para clústeres de Kind, parchea el despliegue para omitir la verificación de TLS:
kubectl patch deployment metrics-server -n kube-system \
  --type='json' -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - En clústeres de Kind, Metrics Server falla sin el argumento `--kubelet-insecure-tls`; parchea el despliegue tras la instalación.
> - Metrics Server solo almacena el punto de datos más reciente, no datos históricos; emplea Prometheus para métricas históricas.

---

### Capítulo 13: Automatización de la resiliencia

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| Sloth | ≥0.11 | Generador de reglas de Prometheus a partir de SLOs |
| Velero | ≥1.12 | Copias de seguridad y recuperación ante desastres en Kubernetes |
| Chaos Mesh | ≥2.6 | Plataforma de ingeniería del caos |

#### Go (requerido para Sloth)

Se requiere Go para instalar Sloth mediante `go install`. Si ya tienes Go instalado, omite este paso.

**macOS:**
```bash
brew install go
```

**Linux (Ubuntu/Debian):**
```bash
# Consulta https://go.dev/doc/install para obtener la versión más reciente
```

**Verificar y configurar PATH:**
```bash
go version
export PATH=$PATH:$(go env GOPATH)/bin
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - El comando `go install` deposita binarios en `~/go/bin/` por defecto. Si `which sloth` devuelve "not found" tras instalar Sloth, tu directorio bin de Go no está incluido en el PATH.
> - Añade `export PATH=$PATH:~/go/bin` en tu `~/.zshrc` o `~/.bashrc` para que el cambio sea permanente.

#### Sloth

Genera reglas de registro (*recording*) y alerta para Prometheus a partir de especificaciones de SLO.

**Opción A — mediante Go (recomendado):**
```bash
go install github.com/slok/sloth/cmd/sloth@latest
```

**Opción B — descarga directa de binario (sin necesidad de Go):**

macOS (Apple Silicon):
```bash
curl -L https://github.com/slok/sloth/releases/download/v0.11.0/sloth-darwin-arm64 -o /usr/local/bin/sloth && chmod +x /usr/local/bin/sloth
```

macOS (Intel):
```bash
curl -L https://github.com/slok/sloth/releases/download/v0.11.0/sloth-darwin-amd64 -o /usr/local/bin/sloth && chmod +x /usr/local/bin/sloth
```

Linux:
```bash
curl -L https://github.com/slok/sloth/releases/download/v0.11.0/sloth-linux-amd64 -o /usr/local/bin/sloth && chmod +x /usr/local/bin/sloth
```

**Verificar instalación:**
```bash
sloth version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Especificaciones de SLO con sintaxis errónea en consultas de Prometheus generan reglas que fallan silenciosamente al cargarse.
> - No ejecutar `sloth validate` antes de aplicar reglas hace que los errores solo se detecten cuando Prometheus rechaza la configuración.
> - Fijar objetivos de SLO excesivamente agresivos (por ejemplo, 99.99%) en servicios no críticos genera fatiga de alertas innecesaria.

#### Velero

Copia de seguridad, restauración y migración de recursos de Kubernetes y volúmenes persistentes.

**macOS:**
```bash
brew install velero
```

**Linux (Ubuntu/Debian):**
```bash
curl -L -o velero.tar.gz https://github.com/vmware-tanzu/velero/releases/latest/download/velero-v1.12.0-linux-amd64.tar.gz
tar xzf velero.tar.gz
sudo mv velero-*/velero /usr/local/bin/
```

**Windows:**
```powershell
choco install velero -y
# O descarga desde https://github.com/vmware-tanzu/velero/releases
```

**Verificar instalación:**
```bash
velero version
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Velero respalda recursos de Kubernetes pero no datos de volúmenes persistentes de forma predeterminada; debes configurar snapshots de volúmenes por separado.
> - No probar restauraciones periódicamente deja sin verificar tu estrategia de copias de seguridad y puede fallar en el momento más crítico.
> - Programaciones de copias de seguridad sin políticas de retención saturan el almacenamiento en la nube e incrementan costes innecesariamente.

#### Chaos Mesh

Plataforma de ingeniería del caos nativa de la nube para Kubernetes que permite experimentos de fallo de pods, retardo de red y pruebas de estrés.

**Todas las plataformas (Helm es multiplataforma):**
```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

**Verificar instalación:**
```bash
kubectl get pods -n chaos-mesh
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - Ejecutar experimentos de caos sin selectores de namespace puede afectar a los espacios de nombres del sistema (`kube-system`, `monitoring`) y provocar la caída del clúster.
> - No definir límites de duración en los experimentos hace que un experimento olvidado continúe inyectando fallos indefinidamente.
> - Chaos Mesh requiere acceso de DaemonSet privilegiado; algunos clústeres con políticas de seguridad estrictas bloquean su instalación.
> - **Clústeres de Kind:** Chaos Mesh despliega por defecto 3 réplicas del `controller-manager`, lo cual puede agotar la memoria en un clúster de nodo único. Si los pods quedan en estado `Pending` por `Insufficient memory`, reduce réplicas: `kubectl scale deployment chaos-controller-manager -n chaos-mesh --replicas=1`. Asimismo, elimina namespaces de capítulos anteriores para liberar memoria.

---

### Capítulo 14: Plataformas aumentadas por IA

| Herramienta | Versión | Propósito |
|-------------|---------|-----------|
| LangChain | ≥0.1 | Framework para aplicaciones con LLM |
| Anthropic Claude API | N/A | Proveedor de LLM (requiere clave de API) |
| ChromaDB | ≥0.4 | Base de datos vectorial para embeddings |

#### Librerías de IA/ML en Python

Instala los paquetes de Python para el pipeline de RAG, el sistema multiagente y los componentes de gobernanza de IA:

```bash
cd Ch14
pip3 install -r requirements.txt
# En Linux, añade --break-system-packages si instalas globalmente
```

> [!WARNING]
> **Errores comunes a tener en cuenta**
> - La API de LangChain cambia con frecuencia entre versiones menores; fija tu versión en `requirements.txt` para prevenir cambios incompatibles.
> - El modo en memoria por defecto de ChromaDB pierde todos los embeddings al reiniciar; configura almacenamiento persistente para todo uso más allá de pruebas rápidas.
> - Pueden alcanzarse los límites de tasa de la API de Anthropic al procesar documentos por lotes para RAG; implementa retroceso exponencial y control de frecuencia de peticiones.
> - Los scripts se ejecutan en modo simulado (*mock*) si no hay clave de API. Define `ANTHROPIC_API_KEY` para obtener respuestas reales de LLM.

#### Clave de API de LLM (Opcional)

Todos los scripts del Capítulo 14 se ejecutan en modo simulado de forma predeterminada; no se precisa clave de API. Para emplear un LLM real, configura una de las siguientes opciones:

**macOS / Linux:**
```bash
# Opción A: Anthropic Claude (recomendado)
export ANTHROPIC_API_KEY="sk-ant-tu-clave-aqui"

# Opción B: LLM local con Ollama (sin necesidad de clave)
ollama pull mistral && ollama serve
```

**Windows (PowerShell):**
```powershell
$env:ANTHROPIC_API_KEY = "sk-ant-tu-clave-aqui"
# O configúralo en Propiedades del Sistema > Variables de entorno para persistencia
```

---

## Solución de problemas

### Docker Desktop no inicia

En Windows, asegúrate de que WSL 2 esté habilitado y de que el paquete de actualización del kernel de Linux de WSL 2 esté instalado. En macOS, verifica que haya suficiente espacio en disco y que Rosetta 2 esté instalado en Macs con Apple Silicon (`softwareupdate --install-rosetta`).

### Problemas de red en clústeres de Kind

Si los pods no pueden acceder a redes externas, revisa la configuración de red de Docker. En Linux, asegúrate de que las reglas de `iptables` permitan el reenvío (*forwarding*). En macOS/Windows, incrementa la memoria asignada a Docker Desktop al menos a 4 GB.

### Fallos de instalación de charts de Helm

Ejecuta siempre `helm repo update` antes de instalar charts. Si un chart falla debido a restricciones de recursos, verifica que tu clúster de Kind cuente con suficiente CPU y memoria. Considera crear un clúster de Kind multi-nodo para configuraciones más próximas a producción.

### Conflictos de paquetes en Python

Utiliza entornos virtuales para aislar dependencias por capítulo. Crea uno con:

```bash
python3 -m venv .venv && source .venv/bin/activate
# En Windows: .venv\Scripts\activate
```

Esto evita conflictos entre capítulos que pudieran emplear versiones distintas de un mismo paquete.

### Problemas con el PATH en Windows

Tras instalar herramientas mediante Chocolatey, es posible que debas reiniciar tu terminal para que las modificaciones de PATH surtan efecto. Si un comando no se encuentra, comprueba que la ruta de instalación de la herramienta figure en el PATH de tu sistema.

---

## Referencia rápida: Herramientas por capítulo

La siguiente tabla ofrece una consulta rápida de las herramientas necesarias por capítulo. Un asterisco (*) indica que la herramienta se introduce por primera vez en dicho capítulo.

| Capítulo | Herramientas Requeridas (* = introducida por primera vez) |
|----------|----------------------------------------------------------|
| 1 | Git, Docker, Python/UV, Pulumi*, Bitwarden CLI*, CircleCI CLI*, pre-commit*, pytest |
| 2 | Flux CD*, Istio*, Kustomize*, bats-core*, Kind, kubectl, Helm |
| 3 | Keycloak*, OPA Gatekeeper*, cert-manager* |
| 4 | OpenTelemetry SDK*, Prometheus*, Grafana*, Jaeger*, Loki* |
| 5 | ArgoCD*, Flask, Express.js*, Winston*, OTEL JS SDK* |
| 6 | Backstage*, PostgreSQL |
| 7 | Flask, @kubernetes/client-node*, @octokit/rest* |
| 8 | GitHub Actions, Trivy* |
| 9 | Crossplane* |
| 10 | Yeoman*, Renovate*, Backstage CLI |
| 11 | conftest*, OPA CLI*, pre-commit, prometheus-client* |
| 12 | OpenCost*, Karpenter*, VPA*, Metrics Server* |
| 13 | Go*, Sloth*, Velero*, Chaos Mesh* |
| 14 | LangChain*, Anthropic Claude API*, ChromaDB* |

> **Nota:** Todos los capítulos asumen que las herramientas fundamentales (Git, Docker, Python, Node.js, kubectl, Kind y Helm) ya están instaladas. Consulta la sección [Herramientas fundamentales](#herramientas-fundamentales) al comienzo de este apéndice.

**Sitio web complementario:** Para consultar los scripts de instalación más recientes, actualizaciones de versiones y recursos adicionales, visita https://peh-packt.platformetrics.com/

---

**Autor:** Ajay Chankramath (ajay@platformetrics.com)  
**Libro:** The Platform Engineer's Handbook (Packt Publishing)
