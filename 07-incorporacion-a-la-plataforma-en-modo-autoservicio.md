# Parte 2: Mejora de la Productividad a Través de Funciones de Autoservicio

## Capítulo 7: Incorporación a la Plataforma en Modo Autoservicio

Antes de comenzar este capítulo, asegúrate de que los siguientes componentes de los capítulos anteriores estén operativos:
- Clúster de Kind con Argo CD ([Capítulo 2](https://subscription.packtpub.com/book/programming/9781806380138/2)) y la aplicación de demostración del [Capítulo 5](https://subscription.packtpub.com/book/programming/9781806380138/5) desplegada.
- Portal de desarrolladores Backstage con SSO configurado ([Capítulo 6](https://subscription.packtpub.com/book/programming/9781806380138/6)).
- Keycloak con el reino (*realm*) `platform` y al menos un usuario de prueba ([Capítulo 3](https://subscription.packtpub.com/book/programming/9781806380138/3)).
- Token de acceso personal de GitHub con permisos de `repo` y `admin:org` exportado como `GITHUB_TOKEN`.

En el capítulo anterior vimos que el equipo de María desplegó Backstage; sin embargo, la incorporación (*onboarding*) de nuevos equipos todavía requiere intervención manual: creación de *namespaces*, configuración de RBAC y aprovisionamiento de repositorios. En este contexto, se une un nuevo equipo piloto (Team Beta). El proceso manual de incorporación toma entre 3 y 4 días y requiere de 7 a 10 solicitudes (*tickets*). En este capítulo eliminaremos esta fricción para construir una API funcional capaz de aprovisionar un entorno de desarrollo completo en cuestión de minutos.

---

### Creación de una API de Incorporación (*Onboarding API*)

La base de cualquier plataforma de autoservicio es una API bien diseñada que actúe como la única fuente de verdad para todas las operaciones de incorporación. Construir la automatización directamente dentro de la interfaz del portal genera acoplamiento e impide extender las capacidades hacia otros canales. Diseñaremos un servicio de incorporación bajo un enfoque **API-first** que dé soporte por igual al portal web, herramientas CLI, bots de chat y canalizaciones de CI/CD.

#### Ventajas del Enfoque API-first:
- **Evita el bloqueo de interfaz**: Todos los flujos del portal consumen la misma API central.
- **Habilita flujos por CLI**: Los desarrolladores avanzados pueden utilizar `curl` o clientes de línea de comandos tipados.
- **Facilita la automatización**: Las canalizaciones de CI/CD pueden aprovisionar entornos de equipo programáticamente.
- **Admite flujos impulsados por IA**: Agentes y copilotos de IA pueden invocar la API sin duplicar lógica de aprovisionamiento.
- **Fuente única de verdad**: La validación de esquemas, la aplicación de cuotas y el registro de auditoría ocurren en un solo lugar.

#### Flujo de Solicitud de Extremo a Extremo:
1. La solicitud llega vía portal UI, CLI o canalización CI/CD mediante un `POST` a `/api/v1/teams`.
2. `authzService` verifica que el usuario solicitante disponga de los permisos necesarios.
3. `k8sService` aprovisiona el namespace, aplica roles RBAC y configura la cuota de recursos.
4. La API Admin de Keycloak crea los grupos `{team}-admins`, `{team}-developers` y `{team}-viewers`.
5. `gitService` crea el repositorio del equipo en la organización de GitHub.
6. `catalogService` registra la entidad del equipo en el catálogo de Backstage.
7. Se devuelve la respuesta `{ namespace, repo, status: 'provisioning' }` al cliente.

> **Nota**: Los pasos 3 al 6 son idempotentes. Un fallo en el paso 4 puede reintentarse sin necesidad de recrear el namespace.

Extracto de la especificación OpenAPI (`openapi-spec.yaml`):

```yaml
# openapi-spec.yaml (excerpt)
paths:
  /api/v1/teams:
    post:
      summary: Create a new team with provisioned resources
      security:
        - oauth2: [platform:teams:create]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/TeamCreateRequest'
      responses:
        '202':
          description: Team provisioning initiated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProvisioningStatus'
```

> **Figura 7.1** - Arquitectura de la API de incorporación

El patrón **Controlador-Servicio-Repositorio** desacopla la lógica de negocio en capas comprobables y mantenibles:

```javascript
// services/teamService.js
// Dependencies are injected via the constructor — this makes each
// service client explicit and mockable in unit tests.
class TeamService {
  constructor({ authzService, k8sService, gitService, catalogService }) {
    this.authzService = authzService;       // permission check (OPA/RBAC)
    this.k8sService = k8sService;           // Kubernetes namespace, RBAC, quota
    this.gitService = gitService;           // GitHub/GitLab repository creation
    this.catalogService = catalogService;   // Backstage catalog registration
  }

  async createTeam(teamRequest, requestingUser) {
    // 1. Validate the requester has permission
    await this.authzService.requirePermission(
      requestingUser,
      'platform:teams:create'
    );

    // 2. Create namespace — teamRequest.name is the validated team slug
    const namespace = await this.k8sService.createNamespace({
      name: `team-${teamRequest.name}`,
      labels: {
        'platform.io/team': teamRequest.name,
        'platform.io/owner': requestingUser.email,
        'platform.io/created-by': 'onboarding-api',
      },
    });

    // 3. Apply RBAC roles for the tier
    await this.k8sService.applyTeamRBAC(namespace, teamRequest.members);
    await this.k8sService.applyResourceQuota(namespace, teamRequest.tier);

    // 4. Create source repository
    const repo = await this.gitService.createRepository(teamRequest);

    // 5. Register in the Backstage catalog
    await this.catalogService.registerTeam(teamRequest, namespace, repo);

    return { namespace, repo, status: 'provisioning' };
  }
}
```

---

### La Oportunidad de la Incorporación (*The Onboarding Opportunity*)

Las primeras impresiones definen la percepción que los desarrolladores tendrán de la plataforma. La investigación en experiencia de desarrollo sugiere la **"regla de los siete minutos"** [8]: cuando el camino para alcanzar la productividad supera los siete minutos de fricción, los desarrolladores buscan alternativas externas (cuentas personales en la nube, infraestructura en la sombra o desvíos del flujo oficial).

Métricas clave para evaluar la incorporación:
- **Tiempo hasta el primer despliegue (*Time-to-first-deploy*)**.
- **Número de tickets de soporte creados durante el proceso**.
- **Tasa de abandono de la plataforma**.

---

### Automatización de RBAC, Namespaces y Cuotas

Los recursos de Kubernetes se generan y aplican dinámicamente mediante `k8sService` para garantizar consistencia y repetibilidad.

#### 1. Plantilla de Namespace (`templates/namespace-template.yaml`)

```yaml
# templates/namespace-template.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: "{{ .TeamName }}"
  labels:
    platform.io/managed-by: onboarding-api
    platform.io/team: "{{ .TeamName }}"
    platform.io/tier: "{{ .Tier }}"
    platform.io/cost-center: "{{ .CostCenter }}"
    istio-injection: enabled
  annotations:
    backstage.io/team-url: "https://portal.example.com/catalog/default/group/{{ .TeamName }}"
    platform.io/slack-channel: "#{{ .TeamName }}"
```

#### 2. Plantilla de Roles RBAC (`team-rbac.yaml`)

```yaml
# Team RBAC template
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-developer
  namespace: "{{ .Namespace }}"
rules:
  # Developers can read and rotate secrets (create/update) but not delete.
  # Deletion is irreversible — reserve it for team-admin and platform-admin.
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "create", "update", "patch"]
  - resources: ["pods/log", "pods/exec", "pods/portforward"]
    verbs: ["get", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-developers-binding
  namespace: "{{ .Namespace }}"
subjects:
  - kind: Group
    name: "oidc:{{ .TeamName }}-developers"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: team-developer
  apiGroup: rbac.authorization.k8s.io
```

> **Nota de Seguridad**: El verbo `delete` se omite deliberadamente en la regla de secretos para el rol `team-developer`. La eliminación de secretos es irreversible y queda reservada para `team-admin` y `platform-admin`.

> **Figura 7.2** - Jerarquía RBAC para la incorporación de equipos

#### 3. Cuotas de Recursos Escalonadas (*Tiered Resource Quotas*)

```yaml
# Tiered ResourceQuota templates
# tier: starter
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: "{{ .Namespace }}"
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    persistentvolumeclaims: "5"
    services.loadbalancers: "1"
    count/deployments.apps: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: team-limits
  namespace: "{{ .Namespace }}"
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "4Gi"
```

#### Gestión de Errores y Resiliencia en la Incorporación:
- **Fallos de infraestructura en Kubernetes**: Devolver error `503` con el mensaje del servidor de API; el reintento es seguro al no haberse creado recursos huérfanos.
- **Agotamiento de cuota de clúster**: Comprobar capacidad disponible en la validación previa (*pre-flight check*) y responder con `409 Conflict`.
- **Límites de tasa transitorios (*Rate Limits*)**: Implementar reintentos con retroceso exponencial (*exponential backoff*) y fluctuación aleatoria (*jitter*) en llamadas a las APIs de GitHub o Kubernetes.

---

### Habilitación de la Gestión de Equipos en Autoservicio

El modelo de autoservicio permite delegar la administración de miembros y permisos a los propios responsables de los equipos (*team admins*), liberando al equipo de plataforma de tareas operativas rutinarias.

> **Figura 7.3** - Jerarquía de delegación de permisos

#### Integración con la API Admin de Keycloak (`keycloak-groups.py`)

Aprovisionamiento programático de grupos en el proveedor de identidades:

```python
# keycloak-groups.py — Keycloak Admin API integration
# Called by the onboarding API after Kubernetes provisioning
def provision_team_groups(team_name: str, members: list, lead_email: str) -> dict:
    """
    Creates three Keycloak groups per team and assigns members.
    Group names match the OIDC subjects in team-rbac.yaml:
    {team_name}-admins / {team_name}-developers / {team_name}-viewers
    """
    token = get_admin_token() # client_credentials grant via admin-cli
    group_ids = {}
    for suffix in ['admins', 'developers', 'viewers']:
        group_name = f"{team_name}-{suffix}"
        group_ids[suffix] = create_group(token, group_name) # idempotent
    
    # Assign the team lead to the admins group
    add_user_to_group(token, lead_email, group_ids['admins'])
    
    # Assign all members to the developers group
    for email in members:
        add_user_to_group(token, email, group_ids['developers'])
    
    return group_ids
```

---

### Inicialización de Nuevos Proyectos (*Project Bootstrapping*)

Los desarrolladores conciben el trabajo en términos de **proyectos**, no de recursos aislados. Un proyecto integra cinco elementos clave:
1. Repositorio de código fuente.
2. Namespace(s) de despliegue.
3. Canalización de CI/CD.
4. Entrada en el catálogo de software.
5. Andamiaje de documentación (*TechDocs*).

| Parámetro | Ejemplos |
| :--- | :--- |
| **Parámetros obligatorios** | Nombre del proyecto, Equipo propietario, Tipo de plantilla |
| **Parámetros opcionales** | Nivel de recursos (*starter* por defecto), entornos (*dev* por defecto), visibilidad (*internal* por defecto) |
| **Parámetros derivados** | Nombres de namespaces, URL del repositorio, referencias a entidades del catálogo |
| **Reglas de validación** | Unicidad de nombre, verificación de pertenencia al equipo, comprobación de cuota disponible |
| **Parámetros condicionales** | El entorno de producción requiere centro de coste; la plantilla de ML requiere aprobación de cuota GPU |

> **Tabla 7.1** - Parámetros críticos para la inicialización de proyectos

#### Plantilla de Scaffolder en Backstage (`backstage-template.yaml`)

```yaml
# backstage-template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: nodejs-backend-service
  title: Node.js Backend Service
spec:
  owner: platform-team
  type: service
  parameters:
    - title: Service Details
      required: [name, team]
      properties:
        name:
          title: Service Name
          type: string
          pattern: '^[a-z][a-z0-9-]*$'
        team:
          title: Owning Team
          type: string
          ui:field: OwnerPicker
        tier:
          title: Resource Tier
          type: string
          enum: [starter, standard, enterprise]
          default: starter
  steps:
    - id: fetch-template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          team: ${{ parameters.team }}
    - id: call-onboarding-api
      action: http:backstage:request
      input:
        method: POST
        path: /api/proxy/onboarding/projects
        body:
          name: ${{ parameters.name }}
          team: ${{ parameters.team }}
          tier: ${{ parameters.tier }}
          template: nodejs-backend
    - id: publish-repo
      name: Create GitHub repository
      action: publish:github
      input:
        # 'org' is a required parameter — no hardcoded value
        repoUrl: github.com?owner=${{ parameters.org }}&repo=${{ parameters.name }}
        defaultBranch: main
        repoVisibility: private
        
  # Surface direct links after the workflow completes
  output:
    links:
      - title: Source Repository
        url: ${{ steps["publish-repo"].output.remoteUrl }}
        icon: github
      - title: Catalog Entry
        url: ${{ steps["register-catalog"].output.entityRef }}
        icon: catalog
```

> **Figura 7.4** - Flujo de inicialización de proyectos en autoservicio

---

### Servicios de Incorporación Personalizados frente a Herramientas Cloud-Native

| Factor | API Personalizada | Motor de Flujos de Trabajo (Argo/Tekton) | Kratix / Crossplane | Humanitec |
| :--- | :--- | :--- | :--- | :--- |
| **Curva de aprendizaje** | Familiar (REST) | Moderada | Pronunciada | Moderada |
| **Flexibilidad** | Ilimitada | Alta | Alta | Media |
| **Mantenimiento** | Alto | Medio | Medio | Bajo |
| **Alineación con GitOps** | Manual | Nativa | Nativa | Nativa |
| **Límite de complejidad** | Sin límite | DAGs de flujos de trabajo | Composiciones | Alcance de la plataforma |
| **Tiempo hasta el primer valor** | Lento | Medio | Medio | Rápido |
| **Coste** | Horas de ingeniería | Horas de ingeniería | Horas de ingeniería | Licencia + ingeniería |

> **Tabla 7.2** - Comparativa de alternativas para la construcción de servicios de incorporación

---

### Resumen

- **Arquitectura API-First**: Desacopla la lógica de incorporación de la interfaz gráfica, permitiendo que el portal web, la CLI y las canalizaciones de CI/CD compartan la misma lógica de aprovisionamiento.
- **Automatización de primitivas de Kubernetes**: Genera namespaces, roles RBAC y cuotas escalonadas (*starter*, *standard*, *enterprise*) de forma dinámica e idempotente.
- **Delegación de permisos y federación OIDC**: Mapea los grupos de Keycloak directamente a roles en Kubernetes, permitiendo a los líderes de equipo administrar miembros sin recurrir al equipo de plataforma.
- **Inicialización integral de proyectos**: Proporciona repositorios, namespaces, canalizaciones de CI/CD y registros de catálogo en menos de cinco minutos mediante plantillas de Backstage Scaffolder.
- **Reducción radical de la fricción**: Transforma la incorporación de equipos de un proceso de varios días basado en tickets a una experiencia de autoservicio automatizada.

---

### Referencias

- **[1]** *Kubernetes ConfigMap Concepts*. [https://kubernetes.io/docs/concepts/configuration/configmap/](https://kubernetes.io/docs/concepts/configuration/configmap/)
- **[2]** *Swagger UI — API Documentation Tools*. [https://swagger.io/tools/swagger-ui/](https://swagger.io/tools/swagger-ui/)
- **[3]** *Kubernetes LimitRange Policies*. [https://kubernetes.io/docs/concepts/policy/limit-range/](https://kubernetes.io/docs/concepts/policy/limit-range/)
- **[4]** *Tekton Pipelines Documentation*. [https://tekton.dev/](https://tekton.dev/)
- **[5]** *Crossplane — Open Source Control Planes*. [https://www.crossplane.io/](https://www.crossplane.io/)
- **[6]** *Kratix — Framework for Building Platforms*. [https://docs.kratix.io/](https://docs.kratix.io/)
- **[7]** *Humanitec Platform Orchestrator*. [https://humanitec.io/](https://humanitec.io/)
- **[8]** *The First 7 Minutes of Onboarding User Experience*. [https://productled.com/blog/the-first-7-minutes-of-the-onboarding-user-experience](https://productled.com/blog/the-first-7-minutes-of-the-onboarding-user-experience)
