# Parte 1: Diseño, Construcción y Despliegue de la Plataforma de Ingeniería Principal

## Capítulo 2: Entorno de Ejecución de Plataforma Escalable con Kubernetes y Service Mesh

Ahora que se han establecido los patrones y las prácticas para el equipo de plataforma en NewTech, el primer problema a abordar es la falta de estandarización en la infraestructura utilizada por los diferentes equipos. Actualmente, aunque los equipos utilizan microservicios en contenedores, estos servicios se despliegan de formas muy diversas: algunos utilizan tecnologías *serverless*, otros diversas formas de virtualización, e incluso un equipo mantiene su propio clúster de Kubernetes construido desde cero. 

Tampoco existe un estándar para la observabilidad, por lo que si algo falla en producción, resulta muy difícil para los equipos recibir ayuda, lo que provoca interrupciones prolongadas que ocurren con mayor frecuencia de la deseada. Necesitamos proporcionar una plataforma que libere a los equipos de desarrollo de pensar en cómo debe ejecutarse su software y les permita concentrarse en lo que su software está haciendo. 

En este caso, para proporcionar la escalabilidad y la gestión centralizada necesarias, a la vez que se otorga flexibilidad en el enfoque de despliegue según sea necesario, María decide que un entorno de ejecución de Kubernetes gestionado por la plataforma será una opción ideal. Sin embargo, por mucho que desee ofrecer una excelente experiencia de desarrollador (DevEx) a los equipos que utilicen el entorno de ejecución, también debe asegurarse de que cualquier plataforma desplegada sea estable y confiable a medida que crezca y madure.

La mejor manera de lograr esto de forma confiable a escala es mediante la gestión declarativa de la infraestructura a través de un enfoque programático y *API-first*. Intentar gestionar los despliegues y las configuraciones de infraestructura mediante intervenciones manuales genera cuellos de botella en los procesos a medida que los desarrolladores esperan a que se aprovisionen nuevos recursos, así como inestabilidad entre los entornos de producción y no producción debido a la desviación de configuración (*drift*) al no haber una fuente central de verdad para el estado del entorno. Ten siempre presente que, al construir una plataforma, lo haces para los requisitos actuales, pero, como en cualquier buen diseño de software, también te preparas para el estado futuro, anticipando aspectos que no se pueden definir hoy. Por ello, la extensibilidad (otro principio fundamental de las plataformas), la automatización y la portabilidad entre nubes o entornos locales (*on-premises*) se vuelven igualmente importantes.

Al final de este capítulo, seguiremos estos principios para:

- Configurar clústeres de Kubernetes de plataforma en entornos de producción y no producción mediante Infraestructura como Código (IaC) declarativa, todo en tu máquina local.
- Comprender que los entornos de producción y no producción para los usuarios de la plataforma son diferentes de los de la propia plataforma.
- Desplegar extensiones de ejecución de la plataforma, como una malla de servicios (*service mesh*), utilizando GitOps para establecer una conciliación continua del estado de ejecución desplegado.
- Incorporar pruebas rigurosas, seguridad y verificación de políticas en las canalizaciones de despliegue de la plataforma.

---

### La Lógica Detrás del Uso de Kubernetes para el Entorno de Ejecución de la Plataforma

Tal vez te preguntes: *¿Por qué elegiría María Kubernetes? ¿No podríamos mantenerlo más simple y usar tecnologías serverless en la nube, o simplemente desplegar en un servidor web tradicional?* Para comprender esto, observa la Tabla 2.1 con algunas características comparativas entre plataformas y aplicaciones finales:

| Característica | Plataformas | Aplicaciones Finales |
| :--- | :--- | :--- |
| **Propósito** | Da servicio a otros servicios mediante automatización | Entrega valor de negocio |
| **Ciclo de vida** | Gestionado por el Equipo de Ingeniería de Plataformas | Gestionado por Equipos de Producto |
| **Recursos** | Compartidos entre todos los inquilinos (*tenants*) | Aislados |
| **Seguridad en tiempo de ejecución** | Requiere privilegios elevados de *root* | Privilegios mínimos habilitados |
| **Ejemplo** | - Ingress de red<br>- Pila de monitorización<br>- Controladores de CI/CD<br>- Portal de desarrolladores | - Aplicaciones web<br>- APIs<br>- Bases de datos |

> **Tabla 2.1** - Comparación de características entre plataformas y aplicaciones

Comenzando con las técnicas tradicionales, observamos que soluciones como las plataformas basadas en máquinas virtuales (VM) presentan una mayor sobrecarga de recursos y un aprovisionamiento más lento, con una necesidad significativamente mayor de competencias especializadas en redes y técnicas de endurecimiento (*hardening*) de sistemas operativos. Esto puede generar mayores costes y una sobrecarga operativa considerable para los equipos de plataforma o de operaciones, desplazando a menudo la automatización desde el autoservicio hacia flujos de trabajo basados en solicitudes (*tickets*), lo que añade sobrecarga de coordinación e incrementa la probabilidad de sobredimensionamiento o subdimensionamiento de recursos.

En cuanto a las tecnologías *serverless*, son excelentes para equipos que gestionan despliegues individuales; sin embargo, resulta complejo gestionarlas y asegurarlas de forma estandarizada entre múltiples equipos. En el contexto de una plataforma, para maximizar la adopción, es necesario dar soporte a una amplia variedad de arquitecturas de solución. La mayoría de las tecnologías *serverless* están orientadas a cargas de trabajo sin estado (*stateless*) con tiempos de ejecución cortos. Además, la optimización del uso se traslada desde la programación de la plataforma hacia el comportamiento de la propia carga de trabajo, haciendo que los patrones de uso sean menos predecibles y reduciendo la capacidad del equipo de plataforma para moldear la eficiencia de costes entre equipos.

Otra opción en auge son los contenedores como servicio (*Container as a Service*, CaaS). Esto se sitúa un nivel por debajo de *serverless* puro, pero a gran escala, las restricciones de configuración y las extensiones que los proveedores de nube exigen para mantener sus acuerdos de nivel de servicio (SLA) se vuelven problemáticas. Para ofrecer una automatización fluida entre arquitecturas manteniendo controles estrictos de gobernanza y estandarización, el equipo de plataforma requiere un mayor control sobre el propio orquestador de contenedores del que los servicios CaaS han podido proporcionar.

Fuera de estas opciones, reconocemos que Kubernetes no es el único orquestador de contenedores existente. Para una plataforma de ingeniería, requerimos la orquestación de contenedores para automatizar el despliegue, la gestión, la resolución de problemas y el escalado en producción de aplicaciones en contenedores. Entre las principales soluciones destacan:

- **Kubernetes**: Construido en torno a un estado deseado, donde declaras lo que quieres y Kubernetes se asegura de alcanzarlo. Va mucho más allá de la programación de tareas, abarcando descubrimiento de servicios, autoescalado, RBAC, orquestación de almacenamiento y más. La filosofía principal es que el propio clúster actúa como la plataforma y los contenedores son lo que se despliega.
- **Apache Mesos**: Cuando se presentó hace una década, muchos creyeron que era la solución adecuada al ser un gestor de recursos de clúster que abstraía CPU, memoria y recursos en un único grupo. Al estar diseñado como un núcleo para sistemas distribuidos, tenía mérito en centros de datos tradicionales, pero perdió la batalla de los contenedores por ser demasiado genérico y complejo frente a la propuesta directa y orientada de Kubernetes.
- **Docker Swarm**: Con el objetivo puro de orquestar contenedores, permitía declarar servicios y programarlos en nodos. Sin embargo, carecía de los mecanismos de eficiencia y experiencia de desarrollador de Kubernetes. Al convertirse Kubernetes en un proyecto de la CNCF, la balanza se inclinó definitivamente a su favor en la industria.

En la práctica, la mayoría de las organizaciones no operan Kubernetes directamente sobre infraestructura sin procesar (*bare metal*), sino que consumen servicios gestionados como **Amazon EKS**, **Google GKE** o **Azure AKS**. Los patrones de plataforma descritos en este libro se mantienen idénticos: el proveedor de nube gestiona el plano de control mientras el equipo de plataforma construye sobre él los flujos de trabajo de los desarrolladores y las barreras de seguridad.

En nuestra experiencia, Kubernetes se ha consolidado como el sustrato estándar para la ingeniería de plataformas debido a que sus API declarativas, su modelo de extensibilidad y su ecosistema de operadores permiten implementar las capacidades de la plataforma como software en lugar de como procesos manuales.

| Característica | Plataformas | Aplicaciones Finales |
| :--- | :--- | :--- |
| **Propósito** | Proporciona una API estándar para desplegar configuraciones y ajustes en nombre de los equipos | Configuración robusta para controlar ajustes de despliegue, asignación de recursos y más |
| **Ciclo de vida** | API de plano de control restringida para la configuración de la plataforma | Despliegues continuos (*rolling deployments*) para despliegues sin tiempo de inactividad |
| **Recursos** | Empaquetado eficiente (*bin-packing*) en nodos escalables | *Namespaces* por equipo |
| **Seguridad en tiempo de ejecución** | Capacidad de controlar políticas mediante código para redes y tiempo de ejecución | Soporte para cuotas y RBAC |

> **Tabla 2.2** - Cómo Kubernetes habilita las características de una plataforma

---

### Creación de los Entornos de Ejecución de la Plataforma

Ahora que entendemos la pila tecnológica, podemos empezar a crear el código base para la plataforma principal. Al desplegar el repositorio `platform-team-admin`, se debió haber creado un repositorio vacío llamado `platform-core` en tu organización de GitHub.

Clónalo en tu disco local y configura una estructura de carpetas básica similar a la utilizada en el [Capítulo 1](https://subscription.packtpub.com/book/programming/9781806380138/1), con carpetas para `.circleci`, `modules`, `scripts` y `tests`. Ahora configuraremos un proyecto de Pulumi en Python para nuestra infraestructura, pero esta vez permitiendo múltiples pilas (*stacks*). En Pulumi, las pilas controlan despliegues para distintos entornos utilizando configuraciones similares (análogas a los *workspaces* de Terraform), lo que nos permite dar soporte tanto a entornos de no producción como de producción.

---

### Entornos de Producción y No Producción

En el SDLC de proyectos de software existe habitualmente un mecanismo de promoción a través de entornos como `dev`, `QA`, `UAT`, `producción`, etc. Al desplegar nuestra plataforma, el proceso no debe ser distinto. Sin embargo, el significado de "producción" para una plataforma difiere ligeramente del convencional: mientras que el equipo de plataforma necesita su propio conjunto de entornos SDLC, la plataforma de producción debe proporcionar a los equipos de desarrollo todos los entornos necesarios para su propio ciclo de vida.

> **Figura 2.1** - Entornos de ejecución de la plataforma

El entorno de producción del equipo de plataforma es fundamentalmente diferente del de una aplicación. Cada despliegue de infraestructura debe dar soporte a `app-dev`, `app-qa` y `app-prod`.

- **`platform-sandbox`**: Entorno de preproducción del equipo de plataforma donde se desarrollan y prueban nuevas características, desplegando varias veces al día sin exigir estabilidad constante. Permite innovar y experimentar libremente, sin requisitos estrictos de retención de datos.
- **`platform-preview`**: Entorno más estable donde se validan cambios frente a cargas de trabajo reales antes de pasarlos a producción. La configuración es similar a producción pero a menor escala, ejecutando pruebas automatizadas y al menos una aplicación de demostración. Los equipos de desarrollo pueden acceder para probar nuevas funciones o posibles cambios incompatibles (*breaking changes*).
- **Entornos de Aplicación (`app-dev`, `app-qa`, `app-prod`)**: Constituyen la "producción" desde la perspectiva de la plataforma. Son los espacios donde los equipos de producto desarrollan y ejecutan sus aplicaciones, separados lógicamente mediante *namespaces* de Kubernetes, clústeres independientes o entornos efímeros. Todos ellos deben ser estables y mantener configuraciones de seguridad y gobernanza consistentes para evitar sorpresas durante las promociones.

#### Configuración de Múltiples Entornos de Plataforma

Para nuestra plataforma, comenzaremos con un entorno `platform-sandbox` (no producción) y un entorno `app-dev` (producción inicial). Crea dos pilas de Pulumi en el proyecto `platform-core` con la CLI de Pulumi:

```bash
pulumi stack init platform-sandbox
pulumi stack init app-dev
```

Esto creará dos archivos en la raíz del proyecto para almacenar configuraciones por entorno: `Pulumi.platform-sandbox.yaml` y `Pulumi.app-dev.yaml`.

Mantener estos archivos consistentes garantiza que lo probado en no producción funcione igual en producción y facilita la reproducción de incidencias para depuración. Para comenzar, define metadatos del clúster en cada entorno:

```yaml
config:
  app:cluster-info:
    name: pe-sandbox
    kind-image: kindest/node:v1.31.0
    wait-seconds: 60
```

#### Despliegue de la Red Principal

Para lograr un perfil de seguridad y rendimiento ideal, recomendamos configurar una segmentación de red adecuada basada en principios de Confianza Cero (*Zero-Trust*), donde el acceso se deniega por defecto y solo se permiten conexiones explícitas. En nuestro clúster, esto se traduce en utilizar distintas subredes según la función:
- Los balanceadores de carga y puntos de entrada se ubican en una **subred pública** con puertos de escucha definidos.
- Todos los servicios de plataforma y despliegues de equipos se sitúan en una **subred privada** que solo acepta conexiones desde los balanceadores de la subred pública, evitando el acceso directo desde el exterior.

> **Figura 2.2** - Configuración inicial de red de la plataforma

Para nuestra plataforma local, como todos los entornos se desplegarán en la misma máquina dentro del mismo espacio de red de Docker, debemos controlar los rangos CIDR utilizados para evitar solapamientos entre entornos.

> **Consejo**: Antes de asignar rangos CIDR, verifica los rangos de VPN disponibles para evitar conflictos comunes con `10.0.0.0/8`. Una solución práctica es usar `10.0.x.x` para sandbox, `10.1.x.x` para preview, etc. Documenta siempre estas asignaciones.

Para `platform-sandbox`, define los siguientes ajustes de red:

```yaml
app:network:
  dockerNetwork: vpc-platform-sandbox
  vpcCidr: 10.0.0.0/16
  publicCidr: 10.0.1.0/24
  privateCidr: 10.0.2.0/24
  extraPortMappings:
    - hostPort: 8080
      containerPort: 80
      protocol: TCP
    - hostPort: 8443
      containerPort: 443
      protocol: TCP
```

Crea el archivo `modules/network.py` con las *DataClasses* de Python para estructurar la información:

```python
@dataclass
class PortMap:
    hostPort: int
    containerPort: int
    protocol: str = "TCP"

@dataclass
class NetworkConfig:
    dockerNetwork: str
    vpcCidr: str
    podCidr: str
    serviceCidr: str
    extraPortMappings: Optional[List[PortMap]] = None
```

Los valores `podCidr` y `serviceCidr` son espacios de direcciones virtuales utilizados por el CNI de Kubernetes dentro del clúster y no entran en conflicto con la red Docker que proporciona conectividad entre nodos.

A continuación, define las funciones para crear la red de Docker y generar la configuración de Kind:

```python
def ensure_docker_network(cfg: NetworkConfig) -> local.Command:
    """Create (or ensure) the Docker bridge network"""
    return local.Command(
        "docker:net",
        create=f"docker network create {cfg.dockerNetwork} --subnet {cfg.vpcCidr} || true",
        delete=f"docker network rm {cfg.dockerNetwork} || true",
    )

def render_kind_config(cluster_name: str, net: NetworkConfig) -> str:
    """Produce a Kind config YAML bound to the Docker network + CIDRs."""
    node: Dict[str, Any] = {"role": "control-plane"}
    if net.extraPortMappings:
        node["extraPortMappings"] = [vars(pm) for pm in net.extraPortMappings]

    kind_cfg: Dict[str, Any] = {
        "kind": "Cluster",
        "apiVersion": "kind.x-k8s.io/v1alpha4",
        "networking": {
            "podSubnet": net.podCidr,
            "serviceSubnet": net.serviceCidr,
        },
        "nodes": [node, {"role": "worker"}],
    }
    return yaml.safe_dump(kind_cfg, sort_keys=False)

def write_kind_config(cluster_name: str, yaml_content: str) -> local.Command:
    """Write the Kind config YAML to a stable path for Pulumi runs."""
    path = f".pulumi/kind/{cluster_name}.yaml"
    b64 = base64.b64encode(yaml_content.encode("utf-8")).decode("ascii")
    script = (
        "bash -euo pipefail -lc "
        f"'mkdir -p .pulumi/kind\n"
        f'base64 -d > {path} <<"EOF"\n'
        f"{b64}\n"
        "EOF\n'"
    )
    return local.Command(
        "kind:cfg",
        create=script,
        delete=f'rm -f "{path}"',
        triggers=[yaml_content, path],  # re-run when content/path changes
    )
```

En `__main__.py`, conecta el módulo de red:

```python
# Map config -> dataclasses
net_cfg = network.NetworkConfig(
    dockerNetwork=network_obj["dockerNetwork"],
    vpcCidr=network_obj["vpcCidr"],
    podCidr=network_obj["podCidr"],
    serviceCidr=network_obj["serviceCidr"],
    extraPortMappings=[
        network.PortMap(**pm) for pm in network_obj.get("extraPortMappings", [])
    ] or None,
)

# Network scaffolding + kind config
docker_net = network.ensure_docker_network(net_cfg)
kind_yaml = network.render_kind_config(cls_cfg.name, net_cfg)
kind_cfg_file = network.write_kind_config(cls_cfg.name, kind_yaml)
```

#### Despliegue de la Plataforma Principal

Crea `modules/cluster.py` para definir el despliegue del clúster de Kind:

```python
@dataclass
class ClusterConfig:
    name: str
    kind_image: Optional[str] = None
    wait_seconds: int = 60
```

Implementa la función `create_kind_cluster`:

```python
def create_kind_cluster(
    cfg: ClusterConfig,
    cfg_file_path: str,
    docker_network: str,
    depends_on=None,
    replace_triggers: Optional[List[str]] = None,
) -> Tuple[local.Command, local.Command, Provider]:
    create_cmd = (
        f'KIND_EXPERIMENTAL_DOCKER_NETWORK="{docker_network}" '
        "kind create cluster"
        f" --name {cfg.name}"
        f' --config "{cfg_file_path}"'
        f" --image {cfg.kind_image}"
        f" --wait {cfg.wait_seconds}s"
    )
    create_opts = ResourceOptions(depends_on=depends_on if depends_on else None)
    create = local.Command(
        "kind:create",
        create=create_cmd,
        delete=f"kind delete cluster --name {cfg.name}",
        triggers=replace_triggers or [],
        opts=create_opts,
    )
    kubeconfig = local.Command(
        "kind:kubeconfig",
        create=f"kind get kubeconfig --name {cfg.name}",
        opts=ResourceOptions(depends_on=[create]),
    )
    provider = Provider(
        "k8s",
        kubeconfig=kubeconfig.stdout,
        enable_server_side_apply=True,
    )
    return create, kubeconfig, provider
```

Llama al módulo desde `__main__.py`:

```python
cls_cfg = cluster.ClusterConfig(
    name=cluster_info["name"],
    kind_image=cluster_info.get("kind-image"),
    wait_seconds=cluster_info.get("wait-seconds"),
)

# Cluster + k8s provider
create, kubeconfig, k8s = cluster.create_kind_cluster(
    cls_cfg,
    cfg_file_path=f".pulumi/kind/{cls_cfg.name}.yaml",
    docker_network=net_cfg.dockerNetwork,
    depends_on=[docker_net, kind_cfg_file],
    replace_triggers=[kind_yaml, net_cfg.dockerNetwork, cls_cfg.kind_image or ""],
)
```

Puedes probar tu despliegue seleccionando la pila `platform-sandbox` y ejecutando `pulumi up` desde la CLI.

---

### Pruebas de Validación del Despliegue

Para mantener la plataforma estable y confiable, se recomienda tratar la infraestructura con el mismo rigor de pruebas que el software de aplicación, implementando pruebas en todas las fases de la canalización:

> **Figura 2.3** - Canalización de despliegue de plataforma reforzada

La canalización se divide en cuatro etapas principales:
1. **Linting de código**: Se ejecuta inmediatamente tras la confirmación, aplicando convenciones y formato.
2. **Pruebas previas al despliegue**: Validación holística del código de infraestructura.
3. **Despliegue**: Aprovisionamiento y configuración de la infraestructura, actualización de cargas de trabajo de Kubernetes y despliegue de las API de la plataforma.
4. **Pruebas posteriores al despliegue**: Comprobaciones de estado básicas del entorno junto con pruebas operativas, de integración, rendimiento y seguridad.

A diferencia de la pirámide de pruebas tradicional de software, en infraestructura la mayoría de las pruebas requieren que los recursos estén efectivamente desplegados para ser significativas.

Utilizaremos **BATS** (*Bash Automated Testing System*) [6] para crear pruebas estructuradas. Crea el archivo `tests/infrastructure.bats`:

```bash
setup_file() {
    # Run once before all tests
    pulumi stack output kubeconfig --stack "$PULUMI_STACK" > kubeconfig.yaml
    export KUBECONFIG="$PWD/kubeconfig.yaml"
    export DOCKER_NETWORK=$(pulumi stack output dockerNetwork --stack "$PULUMI_STACK")
    export CLUSTER_NAME=$(pulumi stack output clusterName --stack "$PULUMI_STACK")
}

teardown_file() {
    # Cleanup after all tests
    rm -f kubeconfig.yaml
}

@test "docker network exists" {
    run docker network inspect "$DOCKER_NETWORK"
    [ "$status" -eq 0 ]
}

@test "kubernetes cluster is accessible" {
    run kubectl get nodes --no-headers
    [ "$status" -eq 0 ]
    node_count=$(echo "$output" | wc -l)
    [ "$node_count" -ge 1 ]
}
```

Ejecuta las pruebas desde la terminal con `bats tests/infrastructure.bats`.

---

### Creación de la Canalización de Despliegue

En `.circleci/config.yml`, declara el ejecutor de máquina local:

```yaml
executors:
  local-machine:
    machine: true
    resource_class: peh_runner/local_laptop_runner
    working_directory: ~/project
```

Define los activadores de la canalización para aplicar desarrollo basado en la rama principal:

```yaml
on-push-main: &on-push-main
  branches:
    only: /^main$/
  tags:
    ignore: /.*/
on-tag-main: &on-tag-main
  branches:
    ignore: /.*/
  tags:
    only: /.*/
```

Configura los flujos de trabajo para `sandbox` (en cada confirmación) y para producción (solo en `tags`):

```yaml
workflows:
  preview:
    jobs:
      - pulumi-preview:
          context: *context
          pulumi_stack: sandbox
          filters: *on-push-main
      - pulumi-update:
          context: *context
          pulumi_stack: platform-sandbox
          requires:
            - pulumi-preview
          filters: *on-push-main

  update:
    jobs:
      - pulumi-preview:
          context: *context
          pulumi_stack: platform-sandbox
          filters: *on-tag-main
      - pulumi-update:
          name: "Deploy to app-dev"
          context: *context
          pulumi_stack: app-dev
          requires:
            - Validate platform-sandbox
          filters: *on-tag-main
```

Añade los trabajos de control de calidad y linting:

```yaml
lint-code:
  description: Code linting
  steps:
    - run:
        name: Check code formatting with black
        command: |
          uv run black --check --diff main.py modules/
    - run:
        name: Type check with mypy
        command: ...
    - run:
        name: Check import sorting with isort
        command: ....
static-analysis: ....
security-scan: ....
```

Entre las herramientas recomendadas para escaneo y seguridad destacan **SonarQube** [9], **Checkmarx**, **Veracode**, **Fortify**, así como herramientas de código abierto como **Semgrep** [10] y **Trivy** [11].

> **Ejercicio: Crear un entorno `app-prod`**
> 
> Añade una tercera pila, `app-prod`, a tu base de código con su archivo de parámetros correspondiente. A continuación, crea un paso de aprobación manual en la canalización antes del despliegue en producción:
> 
> ```yaml
> - approve prod:
>     name: "Approve prod deploy"
>     type: approval
>     requires:
>       - Validate nonprod
>     filters: *on-tag-main
> ```

---

### Activar Servicios de GitOps en la Plataforma

Para mantener una separación clara de responsabilidades entre la infraestructura de la plataforma y el despliegue de aplicaciones, desplegamos un controlador de GitOps (como Flux o Argo CD) que supervisa y concilia continuamente los cambios de configuración.

#### Arquitectura Modular de GitOps

El patrón **App of Apps** [14] centraliza la gestión de configuración mientras preserva la autonomía de los equipos. Un único repositorio (`platform-gitops`) actúa como concentrador central que declara qué repositorios debe supervisar el controlador de GitOps para cada entorno.

> **Figura 2.4** - Arquitectura GitOps de la plataforma

Componentes principales:
- **GitOps controller**: Operador (Flux o Argo CD) responsable de la reconciliación continua.
- **`platform-core`**: Contiene los componentes fundacionales para inicializar el controlador.
- **`platform-gitops`**: Catálogo central que declara los repositorios supervisados por entorno.
- **Configuración por entorno**: Declaraciones que asocian aplicaciones a entornos concretos.
- **Repositorios de equipo**: Repositorios de aplicaciones que contienen manifiestos de Kubernetes, Helm charts o capas de Kustomize.

> **IaC frente a Código de Configuración**: Utiliza IaC (Pulumi/Terraform) exclusivamente para aprovisionar clústeres y recursos de nube. Para el ciclo de vida de aplicaciones y componentes internos de Kubernetes, utiliza herramientas nativas como Helm y Kustomize.

#### Creación del Módulo de Flux

En `modules/flux.py`, define la instalación de Flux mediante Helm:

```python
@dataclass
class FluxConfig:
    version: Optional[str] = None

def install(provider: Provider, namespace="flux-system", version: str | None = None):
    return helm.v3.Release(
        "flux2",
        helm.v3.ReleaseArgs(
            chart="flux2",
            version=version,
            repository_opts=helm.v3.RepositoryOptsArgs(
                repo="https://fluxcd-community.github.io/helm-charts",
            ),
            namespace=namespace,
            create_namespace=True,
            values={"installCRDs": True},
        ),
        opts=ResourceOptions(provider=provider),
    )
```

> **Ejercicio: Crear una comprobación de validación de Flux**
> 
> Instala la CLI de Flux y añade una prueba en `infrastructure.bats` ejecutando `flux check` y verificando que incluya el mensaje `"all checks passed"`.

---

### El Repositorio App of Apps (`platform-gitops`)

Para evitar ramas de larga duración, adoptamos una separación de entornos basada en carpetas [3] combinada con capas (*overlays*) de Kustomize.

> **Figura 2.5** - Estructura de carpetas raíz del repositorio platform-gitops

Archivo `kustomization.yaml` en la raíz de cada entorno:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
labels:
  - pairs: { env: sandbox }
includeSelectors: true
includeTemplates: true
resources:
  - tenants
```

> **Figura 2.6** - Carpeta de entorno de platform-gitops

En `tenants/platform-services/platform-services.yaml`, se definen los recursos personalizados (*CRDs*) de Flux:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: platform-services
  namespace: flux-system
spec:
  url: https://github.com/pe-organization/platform-services.git
  ref: { branch: main }
  interval: 1m
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: platform-services
  namespace: flux-system
spec:
  sourceRef: { kind: GitRepository, name: platform-services }
  path: ./environments/platform-sandbox
  interval: 1m
  timeout: 10m0s
  prune: true
  wait: true
```

#### Inicialización (*Bootstrapping*) del Controlador GitOps

En `modules/flux.py`, añade la función `seed_gitops`:

```python
def seed_gitops(
    provider: Provider,
    *,
    namespace="flux-system",
    repo_url: str,
    path: str,
    interval="1m",
    depends_on=None,
):
    create_opts = ResourceOptions(
        provider=provider,
        depends_on=depends_on if depends_on else None
    )
    git = apiextensions.CustomResource(
        "flux-git",
        api_version="source.toolkit.fluxcd.io/v1",
        kind="GitRepository",
        metadata={"name": "platform-gitops", "namespace": namespace},
        spec={"url": repo_url, "interval": interval, "ref": {"branch": "main"}},
        opts=create_opts,
    )
    ks = apiextensions.CustomResource(
        "flux-kustomization",
        api_version="kustomize.toolkit.fluxcd.io/v1",
        kind="Kustomization",
        metadata={"name": "platform-gitops", "namespace": namespace},
        spec={
            "interval": interval,
            "prune": True,
            "wait": True,
            "sourceRef": {"kind": "GitRepository", "name": "platform-gitops"},
            "path": path,
        },
        opts=ResourceOptions(provider=provider, depends_on=[git]),
    )
    return git, ks
```

> **Ejercicio: Crear el almacén de configuración para `app-dev` y `app-production`**
> 
> ```bash
> # Crear estructura de carpetas
> mkdir -p environments/app-dev environments/app-production
> 
> # Inicializar pilas de Pulumi
> pulumi stack init app-dev
> pulumi stack init app-production
> 
> # Validar registro
> pulumi stack ls
> pulumi preview --stack app-dev
> ```

---

### Habilitación de Servicios y Extensiones de la Plataforma

| Característica | Extensiones | Servicios |
| :--- | :--- | :--- |
| **Rol principal** | Extienden la API de Kubernetes para aprovisionar y configurar recursos | Recopilan, analizan y devuelven datos sobre el clúster o sus aplicaciones |
| **API de Kubernetes** | Habilita la creación de nuevos tipos de recursos vía YAML | Proporciona datos u operatividad sobre el clúster y aplicaciones |
| **Alcance de configuración** | El usuario interactúa directamente con la definición del recurso | Aprovisiona recursos totalmente abstraídos de la experiencia de usuario |
| **Propiedad** | Equipo de plataforma | Equipos de dominio |
| **Ejemplos** | Service Mesh, operadores de infraestructura, StorageClasses | Colectores de observabilidad, escaneo de seguridad |

> **Tabla 2.3** - Extensiones frente a Servicios

#### Creación del Repositorio `platform-services`

> **Figura 2.7** - Raíz del repositorio platform-services

La carpeta `base/` contiene las configuraciones comunes de Helm, mientras que las carpetas de entorno contienen los parches específicos (límites de recursos, versiones de imagen, etc.).

---

### Uso de una Malla de Servicios (*Service Mesh*)

Una malla de servicios gestiona la comunicación entre microservicios proporcionando:
- **Seguridad**: Cifrado mTLS automático y autenticación entre servicios.
- **Gestión de tráfico**: Enrutamiento avanzado, reintentos, *circuit breakers* y tiempos de espera.
- **Observabilidad**: Emisión transparente de métricas, trazas y registros sin alterar el código de las aplicaciones.
- **Políticas**: Aplicación centralizada de reglas de autorización.

> **Figura 2.8** - Componentes de la malla de servicios

| Aspecto | Plano de Control (Istiod) | Plano de Datos (Sidecar Envoys) |
| :--- | :--- | :--- |
| **Rol** | Gestión y configuración centralizada | Gestión distribuida de peticiones |
| **Componentes** | Pilot, Citadel y Galley | Contenedores proxy Envoy |
| **Despliegue** | Servicio único | Inyectado en cada pod de aplicación |
| **Gestión de tráfico** | Define reglas de enrutamiento y reintentos | Ejecuta las decisiones de enrutamiento |
| **Seguridad** | Rota certificados y define políticas | Aplica mTLS y valida certificados |
| **Observabilidad** | Configura y agrega ajustes | Emite métricas, registros y trazas |
| **Políticas** | Distribuye reglas de autorización | Las aplica en tiempo de ejecución |
| **Impacto de fallo** | No se pueden añadir nuevas reglas | La aplicación deja de funcionar |

> **Tabla 2.4** - Plano de Control frente a Plano de Datos en el Service Mesh

#### Despliegue de la Malla de Servicios (Istio)

En `environments/base/helm-repos.yaml`:

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: istio
  namespace: flux-system
spec:
  interval: 30m
  url: https://istio-release.storage.googleapis.com/charts
  timeout: 1m
```

En `environments/base/istio-helm.yaml`:

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: istio-base
  namespace: flux-system
spec:
  interval: 5m
  chart:
    spec:
      chart: base
      sourceRef:
        kind: HelmRepository
        name: istio
        namespace: flux-system
  targetNamespace: istio-system
  install:
    createNamespace: true
    timeout: 10m0s
    remediation:
      retries: 3
      remediateLastFailure: true
```

En `environments/platform-sandbox/istio.yaml`:

```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata: {name: istio-base, namespace: flux-system}
spec:
  chart:
    spec:
      version: 1.22.4
```

En `environments/platform-sandbox/kustomization.yaml`:

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base/helm-repos.yaml
  - ../base/istio-helm.yaml
patches:
  - path: istio.yaml
labels:
  - pairs: {env: platform-sandbox}
includeSelectors: true
includeTemplates: true
```

---

### Pruebas de Servicios y Extensiones de Plataforma

> **Figura 2.9** - Flujo de trabajo de pruebas de humo para la malla de servicios

Crea el script `scripts/flux_reconcile.sh` para forzar la sincronización inmediata:

```bash
#!/bin/bash
set -eux

flux reconcile kustomization platform-services -n flux-system --with-source

# wait 30s for it to settle
for i in $(seq 1 30); do
  S=$(flux get kustomization platform -n flux-system -o json | jq -r \
    '.status.conditions[] | select(.type=="Ready") | .status')
  [ "$S" = "True" ] && break || sleep 5
done
```

Define una prueba de Gateway en `tests/gateway_job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-gateway
  namespace: istio-system
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: curl
          image: curlimages/curl:8.10.1
          args:
            - sh
            - -lc
            - |
              set -e
              H="${SMOKE_HOST:platform-sandbox.local}"
              curl -sS -H "Host: $H" --max-time 10 \
                http://istio-system.istio-ingress.svc.cluster.local/ \
                | head -n 3
```

Añade la ejecución y limpieza del trabajo en `scripts/flux_reconcile.sh`:

```bash
#!/bin/bash
set -eux

kubectl -n istio-system apply -f smoke/http-gateway.yaml

# Wait for job completion (120s); print logs on failure
kubectl -n istio-system wait --for=condition=complete \
  job/smoke-gateway --timeout=120s || \
  (kubectl -n istio-system logs job/smoke-gateway; exit 1)

# Remove configuration once complete
kubectl -n istio-system delete -f smoke/http-gateway.yaml
```

---

### Validación de Despliegues con Políticas como Código (*Policy-as-Code*)

Mediante **Rego** con **Open Policy Agent (OPA)** [4] y **conftest** [5], definimos reglas de cumplimiento aplicadas automáticamente en la canalización de CI/CD.

Crea `policy/flux.rego` para denegar imágenes con la etiqueta `:latest`:

```rego
# deny :latest images
deny contains msg if {
    input.kind == "Deployment"
    some i
    endswith(input.spec.template.spec.containers[i].image, ":latest")
    msg := sprintf(
        "Deployment %s uses :latest tag in container %s",
        [input.metadata.name, input.spec.template.spec.containers[i].name],
    )
}
```

Política para restringir despliegues a *namespaces* autorizados:

```rego
# deny deployments to unauthorized namespaces
deny contains msg if {
    input.kind == "Deployment"
    allowed_namespaces := {"app-dev", "app-qa", "app-prod"}
    not allowed_namespaces[input.metadata.namespace]
    msg := sprintf("Deployment %s targets unauthorized namespace: %s", [input.metadata.name, input.metadata.namespace])
}
```

---

### CI/CD con GitOps

> **Figura 2.10** - Flujo de trabajo CI/CD habilitado para GitOps

El flujo se estructura en 9 pasos:
1. El desarrollador crea una rama de características de corta duración y confirma los cambios.
2. Validación previa a la fusión (linting, pruebas unitarias, escaneos de seguridad y comprobación de políticas).
3. Fusión automática a la rama `main`.
4. Creación, etiquetado y publicación de imágenes de contenedor en el registro.
5. Confirmación de manifiestos actualizados en el repositorio de configuración GitOps.
6. Activación de sincronización inmediata en lugar de esperar el intervalo de sondeo.
7. El controlador de GitOps (Flux/Argo CD) aplica los cambios en el clúster.
8. Validación posterior al despliegue (pruebas de humo, integración y rendimiento).
9. Despliegue en producción tras la validación exitosa.

> **Ejercicio: Política para denegar lanzamientos con versiones de Helm no fijadas**
> 
> Crea una política de OPA en Rego que impida despliegues de Helm charts que contengan versiones flotantes o comodines (por ejemplo, `v2.*.*`), integrándola en la canalización antes de confirmar los manifiestos en el repositorio GitOps.

---

### Resumen

En este capítulo, hemos establecido el entorno de ejecución de plataforma fundacional de NewTech mediante la construcción de un entorno de Kubernetes de nivel de producción gestionado mediante infraestructura como código declarativa y principios de GitOps [13]. Implementamos la distinción crucial entre entornos de equipo de plataforma y entornos de aplicaciones de usuario, asegurando un SDLC adecuado para la plataforma y estabilidad para los desarrolladores.

Utilizando Pulumi y Kind, construimos clústeres segmentados en red con principios de Confianza Cero, desplegamos configuraciones mediante pilas de Pulumi y establecimos pruebas de validación con BATS. Activamos los servicios de GitOps con Flux aplicando el patrón *App of Apps* para desacoplar el aprovisionamiento de infraestructura (`platform-core`) de la configuración de aplicaciones (`platform-gitops`).

Desplegamos la malla de servicios Istio para proporcionar cifrado mTLS transparente, gestión de tráfico y observabilidad sin requerir cambios en el código de las aplicaciones. Finalmente, implementamos Políticas como Código con OPA y Rego y estructuramos una canalización de CI/CD completa con reconciliación inmediata.

---

### Referencias

- **[1]** *Flux CD Documentation — GitOps Toolkit*. [https://fluxcd.io/flux/](https://fluxcd.io/flux/)
- **[2]** *Istio Documentation — Service Mesh*. [https://istio.io/latest/docs/](https://istio.io/latest/docs/)
- **[3]** *Kustomize — Kubernetes-native configuration management*. [https://kustomize.io/](https://kustomize.io/)
- **[4]** *Open Policy Agent / Rego Language Reference*. [https://www.openpolicyagent.org/docs/latest/](https://www.openpolicyagent.org/docs/latest/)
- **[5]** *conftest — Policy testing for configuration files*. [https://www.conftest.dev/](https://www.conftest.dev/)
- **[6]** *BATS (Bash Automated Testing System)*. [https://bats-core.readthedocs.io/](https://bats-core.readthedocs.io/)
- **[7]** *Helm — The Kubernetes Package Manager*. [https://helm.sh/docs/](https://helm.sh/docs/)
- **[8]** *CircleCI — CI/CD pipeline documentation*. [https://circleci.com/docs/](https://circleci.com/docs/)

#### Herramientas de Seguridad Referenciadas
- **[9]** *SonarQube — Open source platform for continuous inspection*. [https://www.sonarsource.com/products/sonarqube/](https://www.sonarsource.com/products/sonarqube/)
- **[10]** *Trivy — Open source security scanner*. [https://trivy.dev/](https://trivy.dev/)
- **[11]** *Semgrep — Static analysis tool*. [https://semgrep.dev/](https://semgrep.dev/)

#### Conceptos y Estándares
- **[12]** *Trunk Based Development*. [https://trunkbaseddevelopment.com/](https://trunkbaseddevelopment.com/)
- **[13]** *GitOps Principles (OpenGitOps)*. [https://opengitops.dev/](https://opengitops.dev/)
- **[14]** *App of Apps Pattern (Argo CD Patterns)*. [https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- **[15]** *Envoy Proxy Documentation*. [https://www.envoyproxy.io/docs/](https://www.envoyproxy.io/docs/)
- **[16]** *mTLS in Istio*. [https://istio.io/latest/docs/concepts/security/#mutual-tls-authentication](https://istio.io/latest/docs/concepts/security/#mutual-tls-authentication)
