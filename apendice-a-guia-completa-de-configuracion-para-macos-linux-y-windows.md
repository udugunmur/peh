# Apéndice A: Guía Completa de Configuración para macOS, Linux y Windows

## Guía Integral de Instalación y Configuración del Entorno de Plataforma

Antes de continuar, consulta la Guía de Instalación en el repositorio complementario y en el sitio web complementario en [peh-packt.platformetrics.com](https://peh-packt.platformetrics.com/) para obtener las instrucciones de configuración más actualizadas y completas, ya que las versiones de las herramientas y los requisitos previos pueden haber cambiado desde la publicación.

---

### Configuración del Conjunto de Herramientas Organizacionales

Antes de comenzar a programar, debes asegurarte de que el conjunto de herramientas esté configurado para una organización simulada que nos permita practicar técnicas de ingeniería de plataformas a escala.

#### 1. Configuración de la Cuenta de Pulumi
1. Regístrate para obtener una cuenta gratuita de Pulumi en [pulumi.com](https://www.pulumi.com/).
   *(Nota: Para los ejercicios de este libro no se requiere una cuenta de Organización; una cuenta Individual es suficiente).*
2. Utilizando el enlace de perfil en la esquina superior derecha, crea un **Personal Access Token (PAT)**.
3. Guarda este valor para almacenarlo posteriormente en un gestor de secretos.

> [!WARNING]
> **Errores comunes a evitar:**
> - Olvidar aplicar un backend de estado coherente (local vs. nube) provoca desviaciones (*drift*).
> - Crear múltiples tokens sin registrarlos dificulta su revocación.
> - Omitir salvaguardas de políticas como código permite la entrada de patrones de infraestructura inseguros.

#### 2. Configuración de la Cuenta de GitHub
Recomendamos crear una organización de GitHub dedicada para los ejercicios prácticos de este libro:
1. Crea una nueva organización siguiendo la documentación oficial de GitHub (ej. `acme-peh-org`).
2. Crea un **Fine-Grained Personal Access Token** autorizado para dicha organización con los siguientes permisos:
   - Acceso a todos los repositorios de la organización.
   - **Administration**: Read & Write.
   - **Commit statuses**: Read & Write.
   - **Contents**: Read & Write.
   - **Custom Properties**: Read & Write.
   - **Metadata**: Read-Only.
3. Guarda este token para su almacenamiento seguro.

> [!WARNING]
> **Errores comunes a evitar:**
> - No habilitar autenticación multifactor (2FA) en todos los miembros expone la seguridad.
> - Omitir reglas de protección de ramas facilita fusiones accidentales en `main`.
> - Mezclar repositorios personales con los de la organización fragmenta la propiedad y accesos.

#### 3. Configuración de CircleCI
1. Crea una cuenta gratuita en [circleci.com](https://circleci.com/).
2. En la página principal, crea una nueva organización con el mismo nombre que tu organización de GitHub.
3. En **Organization Settings**, configura la conexión VCS a todos los repositorios de tu organización de GitHub.
4. En **Organization Settings → Self-hosted runners**, acepta los términos para utilizar ejecutores locales (configuraremos el *runner* más adelante utilizando Kind).

> [!WARNING]
> **Errores comunes a evitar:**
> - Depender exclusivamente de la interfaz gráfica en lugar de YAML versionado en Git impide la reproducibilidad.
> - Agotar la capa gratuita rápidamente al no optimizar los flujos de ejecución.
> - Ejecutores autoalojados mal configurados consumen recursos locales y generan compilaciones inestables.

#### 4. Configuración de Bitwarden
1. Crea una cuenta gratuita en [bitwarden.com](https://bitwarden.com/).
2. Registra de forma segura la contraseña maestra de Bitwarden.
3. Genera una **API Key** desde el portal de Bitwarden y guarda el `client_id` y `client_secret` para su exportación como variables de entorno.

> [!WARNING]
> **Errores comunes a evitar:**
> - Mantener secretos en archivos de configuración planos en lugar de migrarlos al almacén seguro.
> - Políticas de acceso laxas (todos los usuarios ven todos los secretos) vulneran normativas de cumplimiento.
> - Extraer secretos incorrectamente hacia las canalizaciones provoca que se impriman en texto claro en los logs.

---

### Herramientas Fundamentales

Estas herramientas se utilizan a lo largo de múltiples capítulos y deben instalarse en primer lugar.

#### Gestores de Paquetes

- **macOS (Homebrew):**
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

- **Windows (Chocolatey):**
Ejecuta PowerShell como Administrador:
```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString("https://community.chocolatey.org/install.ps1"))
```

- **Linux (Ubuntu/Debian apt):**
```bash
sudo apt update && sudo apt upgrade -y
```

---

#### Git

- **macOS:**
```bash
brew install git
```
- **Linux (Ubuntu/Debian):**
```bash
sudo apt install -y git
```
- **Windows:**
```powershell
choco install git -y # o bien: winget install Git.Git
```
- **Verificación:**
```bash
git --version
```

> [!WARNING]
> Configura siempre `user.name` y `user.email` de forma global para evitar autorías anónimas o incorrectas. Configura reglas de `.gitignore` estrictas para prevenir fugas de secretos y archivos de entorno.

---

#### Docker

- **macOS:**
```bash
brew install --cask docker # Luego inicia Docker Desktop desde Aplicaciones
```
- **Linux (Ubuntu/Debian):**
```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER # Cierra sesión y vuelve a entrar tras este paso
```
- **Windows:**
```powershell
choco install docker-desktop -y # Inicia Docker Desktop y habilita el backend de WSL 2
```
- **Verificación:**
```bash
docker --version
docker compose version
```

> [!TIP]
> Aumenta la memoria asignada a Docker Desktop a un mínimo de 4 GB para soportar la ejecución simultánea de clústeres Kind y múltiples charts de Helm.

---

#### Python 3.10+ y UV

- **macOS:**
```bash
brew install python@3.12 uv
```
- **Linux (Ubuntu/Debian):**
```bash
sudo apt install -y python3 python3-pip python3-venv
pip install uv --break-system-packages
```
- **Windows:**
```powershell
choco install python --version=3.12 -y
pip install uv
```
- **Verificación:**
```bash
python3 --version
uv --version
```

---

#### Node.js 18+ y npm

- **macOS:**
```bash
brew install node@18
```
- **Linux (Ubuntu/Debian):**
```bash
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```
- **Windows:**
```powershell
choco install nodejs-lts -y
```
- **Verificación:**
```bash
node --version
npm --version
```

---

#### kubectl

- **macOS:**
```bash
brew install kubectl
```
- **Linux (Ubuntu/Debian):**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
- **Windows:**
```powershell
choco install kubernetes-cli -y
```
- **Verificación:**
```bash
kubectl version --client
```

---

#### Kind (Kubernetes in Docker)

- **macOS:**
```bash
brew install kind
```
- **Linux (Ubuntu/Debian):**
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
```
- **Windows:**
```powershell
choco install kind -y
```
- **Verificación:**
```bash
kind --version
```

---

#### Helm

- **macOS:**
```bash
brew install helm
```
- **Linux (Ubuntu/Debian):**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
- **Windows:**
```powershell
choco install kubernetes-helm -y
```
- **Verificación:**
```bash
helm version
```

---

### Instrucciones de Instalación Específicas por Capítulo

#### Capítulo 1: Sentando las Bases

| Herramienta | Versión Mínima | Propósito |
| :--- | :--- | :--- |
| **Pulumi** | ≥3.0 | Motor de Infraestructura como Código |
| **Bitwarden CLI** | ≥2024.1 | Gestión de secretos |
| **CircleCI CLI** | Latest | Gestión de canalizaciones CI/CD |
| **pre-commit** | Latest | Framework de hooks de Git para calidad de código |
| **pytest** | ≥7.0 | Pruebas unitarias en Python |

- **Pulumi CLI:**
  - *macOS:* `brew install pulumi/tap/pulumi`
  - *Linux:* `curl -fsSL https://get.pulumi.com | sh && echo "export PATH=$HOME/.pulumi/bin:$PATH" >> ~/.bashrc && source ~/.bashrc`
  - *Windows:* `choco install pulumi -y`
  - *Verificación y login:* `pulumi version && pulumi login`

- **Bitwarden CLI:**
  - *macOS:* `brew install bitwarden-cli`
  - *Linux:* `sudo snap install bw`
  - *Windows:* `choco install bitwarden-cli -y`
  - *Verificación:* `bw --version`

- **CircleCI CLI:**
  - *macOS:* `brew install circleci`
  - *Linux:* `curl -fLSs https://raw.githubusercontent.com/CircleCI-CLI/circleci-cli/main/install.sh | bash`
  - *Windows:* Descargar binario desde GitHub Releases y añadir a PATH.
  - *Verificación:* `circleci version && circleci setup`

- **pre-commit:**
  - *macOS:* `brew install pre-commit`
  - *Linux:* `pip install pre-commit --break-system-packages`
  - *Windows:* `pip install pre-commit`
  - *Verificación:* `pre-commit --version`

- **Dependencias Python de Pruebas:**
```bash
pip install pytest pytest-cov pyyaml # En Linux, añadir --break-system-packages
```

---

#### Capítulo 2: Entorno de Ejecución Escalable

| Herramienta | Versión Mínima | Propósito |
| :--- | :--- | :--- |
| **Flux CD** | ≥2.0 | Entrega continua GitOps |
| **Istio** | ≥1.20 | Service mesh con mTLS |
| **Kustomize** | ≥5.0 | Gestión de configuraciones declarativas |
| **bats-core** | ≥1.10 | Pruebas automatizadas de Bash |

- **Flux CD CLI:**
  - *macOS:* `brew install fluxcd/tap/flux`
  - *Linux:* `curl -s https://fluxcd.io/install.sh | sudo bash`
  - *Windows:* `choco install flux -y`
  - *Verificación:* `flux --version && flux check --pre`

- **Istio (istioctl):**
  - *macOS:* `brew install istioctl`
  - *Linux:* `curl -L https://istio.io/downloadIstio | sh - && sudo mv istio-*/bin/istioctl /usr/local/bin/`
  - *Windows:* Descargar de GitHub Releases y agregar a PATH.
  - *Instalación de demo en el clúster:* `istioctl install --set profile=demo -y`

- **Kustomize:**
  - *macOS:* `brew install kustomize`
  - *Linux:* `curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash && sudo mv kustomize /usr/local/bin/`
  - *Windows:* `choco install kustomize -y`
  - *Verificación:* `kustomize version`

- **bats-core:**
  - *macOS:* `brew install bats-core`
  - *Linux:* `sudo apt install -y bats`
  - *Windows:* `npm install -g bats`
  - *Verificación:* `bats --version`

---

#### Capítulo 3: Asegurando el Acceso a la Plataforma

| Herramienta | Versión Mínima | Propósito |
| :--- | :--- | :--- |
| **Keycloak** | ≥22.0 | Gestión de identidad y acceso (OIDC/OAuth2) |
| **OPA Gatekeeper**| ≥3.14 | Controlador de admisión para políticas |
| **cert-manager** | ≥1.13 | Gestión automatizada de certificados TLS |

- **Keycloak (Despliegue Helm):**
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install keycloak bitnami/keycloak \
  --namespace keycloak --create-namespace \
  --set auth.adminUser=admin \
  --set auth.adminPassword=admin
```

- **OPA Gatekeeper (Despliegue Helm):**
```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system --create-namespace \
  --set enableGenerateViolationEvents=true \
  --set constraintViolationsLimit=1000 \
  --set auditIntervalSeconds=60
```

- **cert-manager (Despliegue Helm):**
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

---

#### Capítulo 4: Integración de la Observabilidad

| Herramienta | Versión Mínima | Propósito |
| :--- | :--- | :--- |
| **OpenTelemetry SDK** | Latest | Librerías de instrumentación |
| **Prometheus** | ≥2.45 | Recolección y almacenamiento de métricas |
| **Grafana** | ≥10.0 | Visualización y cuadros de mando |
| **Jaeger** | ≥1.50 | Backend de trazado distribuido (opcional) |
| **Loki** | ≥2.9 | Agregación de logs (opcional) |

- **Stack Prometheus & Grafana (kube-prometheus-stack):**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

- **Jaeger (Opcional):**
```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm install jaeger jaegertracing/jaeger \
  --namespace observability --create-namespace
```

- **Loki (Opcional):**
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace observability --set grafana.enabled=false
```

- **Librerías Python de OpenTelemetry:**
```bash
pip install opentelemetry-api opentelemetry-sdk \
  opentelemetry-exporter-otlp \
  opentelemetry-exporter-prometheus \
  prometheus-client flask
```

---

#### Capítulo 5: Evaluación de la Experiencia de Usuario

| Herramienta | Versión Mínima | Propósito |
| :--- | :--- | :--- |
| **ArgoCD** | ≥2.9 | Entrega continua declarativa GitOps |
| **Flask** | ≥2.3 | Framework web Python (App demo) |
| **Express.js** | ≥4.18 | Framework web Node.js (Demo de instrumentación) |
| **OpenTelemetry JS** | ≥1.17 | SDK de instrumentación en Node.js |
| **Winston** | ≥3.11 | Librería de logging para Node.js |

- **Instalación de ArgoCD en el clúster y CLI:**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
  - *macOS:* `brew install argocd`
  - *Linux:* `curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 && sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd`
  - *Windows:* `choco install argocd-cli -y`

- **Dependencias Node.js de la Aplicación Demo:**
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

#### Capítulo 6: Despliegue y Curación del Portal Backstage

| Herramienta | Versión Mínima | Propósito |
| :--- | :--- | :--- |
| **Backstage** | ≥1.20 | Portal de desarrolladores |
| **PostgreSQL** | ≥14.0 | Base de datos persistente para Backstage |

- **Despliegue vía Helm o Creación Local:**
```bash
# Opción 1: Despliegue con Helm
helm repo add backstage https://backstage.github.io/charts
helm install backstage backstage/backstage --namespace backstage --create-namespace

# Opción 2: Instancia de desarrollo local
npx @backstage/create-app@latest
```

---

#### Capítulo 7: Incorporación a la Plataforma en Autoservicio

- **Dependencias Python del API de Onboarding:**
```bash
pip install flask pyyaml kubernetes
```
- **Dependencias Node.js del Servicio de Provisión:**
```bash
npm install @kubernetes/client-node @octokit/rest
npm install --save-dev jest supertest
```

---

#### Capítulo 8: CI/CD como Servicio de Plataforma

- **Trivy (Escáner de Vulnerabilidades):**
  - *macOS:* `brew install trivy`
  - *Linux:*
```bash
sudo apt install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | \
sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt install -y trivy
```
  - *Windows:* `choco install trivy -y`
  - *Verificación:* `trivy --version`

---

#### Capítulo 9: Gestión de Infraestructura con Crossplane

- **Instalación de Crossplane en el clúster:**
```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace
```
- **Crossplane CLI:**
  - *macOS:* `brew install crossplane/tap/crossplane`
  - *Linux:* `curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh && sudo mv crossplane /usr/local/bin/`

---

#### Capítulo 10: Publicación de Starter Kits

- **Yeoman CLI:** `npm install -g yo`
- **Renovate CLI:** `npm install -g renovate` *(o instalar la GitHub App oficial de Renovate)*

---

#### Capítulo 11: Validación del Cumplimiento y Políticas como Código

- **conftest:**
  - *macOS:* `brew install conftest`
  - *Linux:* `wget https://github.com/open-policy-agent/conftest/releases/latest/download/conftest_Linux_x86_64.tar.gz && tar xzf conftest_Linux_x86_64.tar.gz && sudo mv conftest /usr/local/bin/`
  - *Windows:* `choco install conftest -y`
  - *Verificación:* `conftest --version`

- **OPA CLI:**
  - *macOS:* `brew install opa`
  - *Linux:* `curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64 && chmod +x opa && sudo mv opa /usr/local/bin/`
  - *Windows:* `choco install opa -y`
  - *Verificación:* `opa version`

---

#### Capítulo 12: Optimización de Costes, Rendimiento y Escalabilidad

- **OpenCost (Despliegue Helm):**
```bash
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm repo update
helm install opencost opencost/opencost \
  --namespace opencost --create-namespace \
  --set opencost.prometheus.internal.serviceName="prometheus-server" \
  --set opencost.prometheus.internal.namespaceName="monitoring"
```

- **Karpenter (Helm en AWS EKS):**
```bash
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "${KARPENTER_VERSION}" \
  --namespace karpenter \
  --create-namespace \
  --set settings.aws.clusterName=my-cluster
```

- **Kyverno (Despliegue Helm):**
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno \
  --namespace kyverno --create-namespace
```

- **Vertical Pod Autoscaler (VPA):**
```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh # En Windows ejecutar desde Git Bash o WSL
```

- **Metrics Server:**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

#### Capítulo 13: Automatización de la Resiliencia

- **Sloth CLI:**
  - *macOS:* `brew install slok/sloth/sloth`
  - *Linux:* `curl -L -o sloth https://github.com/slok/sloth/releases/latest/download/sloth-linux-amd64 && chmod +x sloth && sudo mv sloth /usr/local/bin/`
  - *Verificación:* `sloth version`

- **Velero CLI:**
  - *macOS:* `brew install velero`
  - *Linux:* `curl -L -o velero.tar.gz https://github.com/vmware-tanzu/velero/releases/latest/download/velero-v1.12.1-linux-amd64.tar.gz && tar xzf velero.tar.gz && sudo mv velero-*/velero /usr/local/bin/`
  - *Windows:* `choco install velero -y`
  - *Verificación:* `velero version`

- **Chaos Mesh (Despliegue Helm):**
```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

---

#### Capítulo 14: Plataformas Agénticas y Aumentadas por IA

- **Instalación de Librerías de IA y RAG:**
```bash
pip install langchain langchain-openai langchain-community langchain-anthropic \
  chromadb sentence-transformers \
  openai tiktoken \
  prometheus-client kubernetes pyyaml
```

- **Configuración de Variables de Entorno de API Keys:**
  - *macOS / Linux:*
```bash
export OPENAI_API_KEY="sk-your-api-key-here"
export ANTHROPIC_API_KEY="your-anthropic-key-here"
```
  - *Windows (PowerShell):*
```powershell
$env:OPENAI_API_KEY = "sk-your-api-key-here"
$env:ANTHROPIC_API_KEY = "your-anthropic-key-here"
```

---

### Solución de Problemas Frecuentes (*Troubleshooting*)

- **Docker Desktop no inicia**: En Windows, confirma que WSL 2 esté habilitado y actualizado. En macOS con Apple Silicon, asegúrate de haber instalado Rosetta 2 (`softwareupdate --install-rosetta`).
- **Problemas de Red en Kind**: Si los pods no alcanzan redes externas, revisa las reglas de reenvío de paquetes (*packet forwarding*) en `iptables` (Linux) o aumenta la memoria de Docker Desktop (macOS/Windows) a 4 GB o más.
- **Fallos en Instalación de Helm Charts**: Ejecuta siempre `helm repo update` antes de instalar. Si los pods fallan por límites de recursos en Kind, desactiva componentes secundarios no requeridos.
- **Conflictos entre Paquetes de Python**: Utiliza entornos virtuales dedicados por capítulo (`python3 -m venv .venv && source .venv/bin/activate`).
- **Rutas de Windows (PATH)**: Reinicia la terminal tras instalar utilidades con Chocolatey para que las nuevas variables de entorno surtan efecto.

---

### Tabla de Referencia Rápida de Herramientas por Capítulo

| Capítulo | Herramientas Requeridas (* = introducida por primera vez) |
| :--- | :--- |
| **1** | Git, Docker, Python/UV, Pulumi*, Bitwarden CLI*, CircleCI CLI*, pre-commit*, pytest |
| **2** | Flux CD*, Istio*, Kustomize*, bats-core*, Kind, kubectl, Helm |
| **3** | Keycloak*, OPA Gatekeeper*, cert-manager* |
| **4** | OpenTelemetry SDK*, Prometheus*, Grafana*, Jaeger*, Loki* |
| **5** | ArgoCD*, Flask, Express.js*, Winston*, OTEL JS SDK* |
| **6** | Backstage*, PostgreSQL |
| **7** | Flask, @kubernetes/client-node*, @octokit/rest* |
| **8** | GitHub Actions, Trivy* |
| **9** | Crossplane* |
| **10** | Yeoman*, Renovate*, Backstage CLI |
| **11** | conftest*, OPA CLI*, pre-commit, prometheus-client* |
| **12** | OpenCost*, Karpenter*, VPA*, Kyverno*, Metrics Server* |
| **13** | Sloth*, Velero*, Chaos Mesh* |
| **14** | LangChain*, OpenAI API*, ChromaDB* |

> **Nota:** Todos los capítulos asumen que las herramientas fundamentales (**Git, Docker, Python, Node.js, kubectl, Kind y Helm**) ya están instaladas y operativas.
>
> **Sitio Web Complementario:** Para obtener los scripts de instalación actualizados y recursos adicionales, visita [https://peh-packt.platformetrics.com/](https://peh-packt.platformetrics.com/)
