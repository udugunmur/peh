# Parte 2: Mejora de la Productividad a Través de Funciones de Autoservicio

## Capítulo 6: Acelerando DevEx: Despliegue y Curación de su Primer Portal de Desarrolladores

En la Parte 1 del libro, vimos cómo el equipo de María sentó las bases y completó el primer despliegue automatizado, resolviendo los problemas del Día 0 y Día 1. Al comenzar la Parte 2, abordaremos los desafíos del Día 2: la localización de documentación, la solicitud de recursos y la asignación clara de propiedad. Con el objetivo fundamental de reducir la sobrecarga cognitiva de los desarrolladores para que puedan enfocarse en aportar valor a los usuarios finales, la reducción de fricción se aplica a los problemas del Día 2 con el mismo rigor.

Entre las diversas alternativas disponibles, la solución más eficaz consiste en estructurar el **portal de desarrolladores** como el Panel Único de Control (*Single Pane of Glass* - SPOG) que unifique las capacidades de la plataforma construidas en los capítulos anteriores. Los portales no son solo paneles de control (*dashboards*), sino la interfaz principal del desarrollador con la plataforma.

Sin embargo, los portales son un arma de doble filo: un exceso de funciones genera confusión, mientras que muy pocas transmiten una sensación de irrelevancia. En este capítulo analizaremos cómo seleccionar el conjunto inicial de capacidades y curarlas cuidadosamente para sincronizar la evolución del portal con la madurez de la plataforma.

El mayor error en la industria es confundir el portal con la plataforma misma. Una **plataforma** proporciona capacidades fundamentales: canalizaciones de CI/CD, automatización de infraestructura, gestión de secretos y backends de observabilidad. Un **portal** las expone y orquesta: es la interfaz unificada a través de la cual los desarrolladores descubren, usan y miden esas capacidades en un solo lugar.

| Reside en el Backend de la Plataforma | Se Muestra a Través del Portal |
| :--- | :--- |
| Canalizaciones de CI/CD, sistemas de compilación | Estado de canalizaciones y activadores de despliegue |
| Clústeres de Kubernetes, infraestructura | Estado de salud de servicios, paneles de Kubernetes |
| Pila de observabilidad (OTel, Grafana) | Métricas de servicio, paneles de SLO |
| Gestión de secretos (Vault, SOPS) | Flujos de trabajo de rotación de secretos |
| Proveedor de identidad (Keycloak) | Inicio de sesión SSO, control de acceso basado en roles |

> **Tabla 6.1** - Qué reside en el backend de la plataforma frente a qué se expone a través del portal

En este capítulo no añadiremos infraestructura nueva, sino que expondremos las capacidades ya construidas: el proveedor de identidad Keycloak ([Capítulo 3](https://subscription.packtpub.com/book/programming/9781806380138/3)) servirá como base de SSO; la aplicación de demostración ([Capítulo 5](https://subscription.packtpub.com/book/programming/9781806380138/5)) se convertirá en la primera entrada del catálogo; y los flujos GitOps ([Capítulo 2](https://subscription.packtpub.com/book/programming/9781806380138/2)) permitirán automatizar el registro en el catálogo.

---

### Selección y Despliegue de un Portal de Desarrolladores

En lugar de basar la elección del portal en opiniones o presentaciones comerciales, recomendamos evaluar las opciones a través de un marco estructurado en dimensiones medibles: planos de la plataforma, capacidades, perfiles de usuario y resultados de negocio.

> **Figura 6.1** - Marco de toma de decisiones para portales de desarrolladores

Un portal desplegado antes de que existan las capacidades subyacentes de la plataforma se convierte en un **"portal del peligro"** (*portal-of-peril*), caracterizado por menús vacíos, integraciones rotas y frustración del desarrollador. El portal es una capa de integración: no reemplaza herramientas operativas consolidadas como PagerDuty, Jira o Grafana, sino que centraliza su visibilidad.

| Plano de la Plataforma | Qué Abarca | Preguntas Clave |
| :--- | :--- | :--- |
| **Catálogo de Servicios y Propiedad** | Fuente única de verdad para sistemas y equipos | ¿Pueden los desarrolladores encontrar y comprender todos los servicios? ¿Es explícita la propiedad? |
| **Golden Paths y Creación de Plantillas (*Scaffolding*)** | Plantillas estandarizadas para crear nuevos servicios | ¿Pueden los desarrolladores crear nuevos servicios sin depender del conocimiento informal (*tribal*)? |
| **Operaciones y Confiabilidad** | Madurez de SLO, gestión de incidentes, alineación con SRE | ¿Muestra el portal el estado operativo y los manuales de procedimientos (*runbooks*)? |
| **Seguridad y Gobernanza** | Aplicación de políticas, marcos de cumplimiento (SOC 2, PCI, HIPAA) | ¿Pueden los equipos de seguridad auditar y aplicar estándares a través del portal? |
| **Observabilidad y Telemetría** | Integración con monitorización, registros y trazas | ¿Enlaza el portal los servicios con sus paneles de observabilidad correspondientes? |
| **SEI e Información Aumentada con IA** | Inteligencia de ingeniería de software, copilotos de IA | ¿Admite el portal analíticas, predicciones y automatización? |

> **Tabla 6.2** - Criterios de evaluación para portales de desarrolladores

Utilizaremos **Backstage OSS** [1] como referencia debido a su adopción generalizada, su ecosistema de plugins extensible, su ausencia de restricciones de licencia y su integración directa con nuestra pila (Keycloak, Argo CD y OpenTelemetry).

---

### Flujo de Selección

El proceso de selección consta de cuatro etapas secuenciales:

1. **Definir el impulsor principal**: Determinar si el reto prioritario radica en DevEx (herramientas fragmentadas), Operaciones (descubrimiento de manuales e incidentes), Gobernanza (cumplimiento y precisión del catálogo) o Analíticas de IA.
2. **Evaluar la preparación de la plataforma**: Verificar que las capacidades del backend existan antes de exponerlas en la interfaz.
3. **Evaluar el tamaño y madurez del equipo**: Equipos pequeños con capacidad limitada de operaciones de plataforma obtienen valor más rápido con soluciones COTS o gestionadas (Port, OpsLevel). Equipos con experiencia sólida en TypeScript/Frontend obtienen máxima flexibilidad con Backstage OSS.
4. **Mapear a las opciones**: Backstage OSS para control total y aprendizaje profundo; COTS para rapidez de adopción; híbrido (Spotify Portal, Roadie) para Backstage gestionado con soporte empresarial.

> **Figura 6.2** - Ciclo de vida de decisión recomendado para la selección de portales

#### Portales de Promesa y Peligro

El fenómeno del "portal zombi" surge cuando se despliega una interfaz con entusiasmo inicial pero los datos quedan desactualizados, las entradas del catálogo pierden propietarios reales y los menús se saturan de plugins sin uso.

Indicadores clave de éxito de un portal:
- Colaboradores activos actualizando entradas del catálogo semanalmente.
- Crecimiento sostenido en la completitud del catálogo.
- El portal es el primer recurso consultado para descubrimiento de servicios (en lugar de Slack o wikis).
- Recepción de nuevas solicitudes de características por parte de los desarrolladores.

Para recuperar un portal degradado: eliminar implacablemente funciones no utilizadas, ejecutar ciclos de limpieza del catálogo y realizar sesiones de escucha activa con los desarrolladores.

---

### Arquitectura de Backstage

Backstage está construido con **React** en el frontend, **Node.js** en el backend y una base de datos relacional conectada mediante una arquitectura modular de plugins.

> **Figura 6.3** - Vista arquitectónica de Backstage (Cortesía [3])

- **Frontend React**: Servido típicamente en el puerto `3000`.
- **Backend Node.js API**: Ejecutado en el puerto `7007`, gestiona el procesamiento del catálogo, autenticación y APIs de plugins.
- **Base de Datos Relacional**: Se recomienda **PostgreSQL** para persistencia en entornos de producción en lugar de SQLite en memoria.

Configuración de persistencia y red en `app-config.yaml`:

```yaml
# app-config.yaml — persistence and network configuration
app:
  baseUrl: https://portal.${DOMAIN}
backend:
  baseUrl: https://portal.${DOMAIN}
  listen:
    port: 7007
    host: 0.0.0.0
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
```

---

### Navegación y Permisos Basados en Roles

Integrar inicio de sesión único (SSO) permite abandonar el modo invitado y proteger el acceso a la plataforma.

Configuración de integración OIDC con Keycloak en `app-config.yaml`:

```yaml
# app-config.yaml - Keycloak OIDC integration
auth:
  environment: production
  session:
    secret: ${AUTH_SESSION_SECRET} # required for session cookies
  providers:
    oidc: # Backstage has no native 'keycloak' provider
      production:
        metadataUrl: ${KEYCLOAK_URL}/realms/platform/.well-known/openid-configuration
        clientId: ${KEYCLOAK_CLIENT_ID}
        clientSecret: ${KEYCLOAK_CLIENT_SECRET}
        prompt: auto
        signIn:
          resolvers:
            - resolver: emailMatchingUserEntityProfileEmail
  signInPage: oidc # directs users to the OIDC login screen
```

Registro del módulo OIDC en el backend de Backstage (`packages/backend/src/keycloak-oidc-module.ts`):

```typescript
// packages/backend/src/keycloak-oidc-module.ts
import { createBackendModule } from '@backstage/backend-plugin-api';
import { authProvidersExtensionPoint } from '@backstage/plugin-auth-node';
import { createOidcProvider } from '@backstage/plugin-auth-backend-module-oidc-provider';

export const keycloakOidcModule = createBackendModule({
  pluginId: 'auth',
  moduleId: 'keycloak-oidc',
  register(reg) {
    reg.registerInit({
      deps: { providers: authProvidersExtensionPoint },
      async init({ providers }) {
        providers.registerProvider({
          providerId: 'oidc',
          factory: createOidcProvider({
            signIn: {
              resolver: async (info, ctx) => {
                const { profile } = info;
                if (!profile.email) throw new Error('Email claim required');
                return ctx.signInWithCatalogUser({
                  filter: {
                    kind: 'User',
                    'spec.profile.email': profile.email,
                  },
                });
              },
            },
          }),
        });
      },
    });
  },
});
```

Estructura organizativa y asignación de propiedad en el catálogo (`org.yaml`):

```yaml
# org.yaml - Organizational structure for Backstage catalog
---
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: platform-team
  description: Platform Engineering Team
spec:
  type: team
  children: []
---
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: pilot-team-alpha
  description: Pilot Program - Team Alpha
spec:
  type: team
  parent: platform-users
  children: []
---
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: maria.garcia
  annotations:
    keycloak.org/id: ${KEYCLOAK_USER_ID}
spec:
  profile:
    displayName: Maria Garcia
    email: maria.garcia@example.com
  memberOf: [pilot-team-alpha]
```

---

### Habilitación Gradual de Capacidades del Portal

El conjunto adecuado de funciones depende de lo que la plataforma pueda respaldar en cada etapa:

| Nivel de Madurez de la Plataforma | Características Recomendadas del Portal |
| :--- | :--- |
| **Inicial** (Infraestructura básica con automatización limitada) | Catálogo de servicios, TechDocs, Búsqueda, SSO / Identidad |
| **En Crecimiento** (CI/CD y Kubernetes estables y observables) | Paneles en página principal, estado de CI/CD, plugin de Kubernetes, plantillas de Scaffolder |
| **En Maduración** (Plataforma adoptada a nivel de organización y guiada por métricas) | Cuadros de mando (*Scorecards*), paneles de SLO, paneles de FinOps, flujos de incorporación basados en PR |
| **Avanzado** (El equipo de plataforma opera a nivel de producto) | Copilotos de IA, analíticas SEI, plugins personalizados, alertas predictivas |

> **Tabla 6.3** - Madurez de la plataforma frente a madurez del portal

#### Diseño de Página Principal Personalizada con la API de Composabilidad

```tsx
// packages/app/src/components/home/custom-homepage.tsx
// Uses PageBlueprint API (Backstage 1.28+) — replaces createPageExtension
import React from 'react';
import { PageBlueprint } from '@backstage/frontend-plugin-api';
import { Grid } from '@material-ui/core';
import { InfoCard, Header, Page, Content } from '@backstage/core-components';

// NOTE: KubernetesStatusCard and ServiceHealthCard are custom components
// wrapping Backstage's standard InfoCard. Build them in your own frontend
// plugin. The full implementation is in custom-homepage.tsx in the
// companion repository.
import { KubernetesStatusCard } from '../kubernetes/KubernetesStatusCard';
import { ServiceHealthCard } from '../serviceHealth/ServiceHealthCard';

const HomepageComponent = () => (
  <Page themeId="home">
    <Header title="Welcome to the Platform Portal" />
    <Content>
      <Grid container spacing={3}>
        <Grid item xs={12}>
          <InfoCard title="Welcome to My IDP">
            Your single pane of glass for the NewTech platform.
          </InfoCard>
        </Grid>
        <Grid item md={6} xs={12}>
          <KubernetesStatusCard title="Pilot Environment Status" />
        </Grid>
        <Grid item md={6} xs={12}>
          <ServiceHealthCard serviceId="demo-app" />
        </Grid>
      </Grid>
    </Content>
  </Page>
);

export const CustomHomepageBlueprint = PageBlueprint.make({
  params: {
    defaultPath: '/',
    loader: async () => <HomepageComponent />,
  },
});
```

Configuración de TechDocs y motor de búsqueda:

```yaml
# app-config.yaml — TechDocs (local/Kind — default for development)
techdocs:
  builder: local
  generator:
    runIn: local
  publisher:
    type: local

# app-config.production.yaml — layered on top for production overrides
techdocs:
  builder: external
  publisher:
    type: awsS3
    awsS3:
      bucketName: ${TECHDOCS_S3_BUCKET}
      region: ${AWS_REGION}

search:
  elasticsearch:
    node: ${ELASTICSEARCH_URL}
    auth:
      username: ${ELASTICSEARCH_USER}
      password: ${ELASTICSEARCH_PASSWORD}
```

Funcionalidades a posponer en la fase MVP inicial:
- Cuadros de mando y modelos de madurez sin estándares formalizados.
- Paneles de costes y FinOps sin la observabilidad previa requerida.
- Copilotos de IA y automatizaciones avanzadas.
- Integraciones profundas de CI/CD antes de estabilizar las canalizaciones.

---

### Publicación de Servicios en el Catálogo de Software

Definición del componente de la aplicación de demostración (`catalog-info.yaml`):

```yaml
# catalog-info.yaml for the demo app from Chapter 5
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: demo-app
  description: Platform demo application with OpenTelemetry instrumentation
  annotations:
    backstage.io/techdocs-ref: dir:.
    backstage.io/source-location: url:https://github.com/peh/demo-app
    argocd/app-name: pilot-demo-app
    prometheus.io/rule: demo-app-alerts
    grafana/dashboard-selector: demo-app
    pagerduty.com/service-id: ${PAGERDUTY_SERVICE_ID}
  tags:
    - nodejs
    - kubernetes
    - opentelemetry
    - pilot-program
  links:
    - url: https://grafana.example.com/d/demo-app
      title: Grafana Dashboard
      icon: dashboard
    - url: https://argocd.platform.${DOMAIN}/applications/pilot-demo-app
      title: ArgoCD Application
      icon: cloud
spec:
  type: service
  lifecycle: production
  owner: pilot-team-alpha
  system: pilot-program
  providesApis:
    - demo-api
  dependsOn:
    - resource:default/postgresql
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: demo-api
  description: Demo application REST API
spec:
  type: openapi
  lifecycle: production
  owner: pilot-team-alpha
  system: pilot-program
  definition:
    $text: ./api/openapi.yaml
```

#### Incorporación de Servicios Basada en Pull Requests con Scaffolder

Plantilla de Scaffolder para registrar servicios existentes sin escribir YAML manualmente (`templates/onboard-service/template.yaml`):

```yaml
# templates/onboard-service/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: onboard-existing-service
  title: Onboard Existing Service to Catalog
  description: Register an existing service in the software catalog
spec:
  owner: platform-team
  type: service
  parameters:
    - title: Service Information
      required: [name, description, owner]
      properties:
        name:
          title: Service Name
          type: string
          pattern: '^[a-z0-9-]+$'
        description:
          title: Description
          type: string
        owner:
          title: Owner Team
          type: string
          ui:field: OwnerPicker
        repoUrl:
          title: Repository URL
          type: string
          ui:field: RepoUrlPicker
  steps:
    - id: fetch-template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
    - id: create-pull-request
      action: publish:github:pull-request
      input:
        repoUrl: ${{ parameters.repoUrl }}
        branchName: onboard-${{ parameters.name }}-to-catalog
        title: 'feat: Onboard ${{ parameters.name }} to developer portal'
        description: |
          This PR adds catalog-info.yaml to register this service in our developer portal.
          
  # The output block surfaces the PR URL to the developer
  # immediately after the scaffolder workflow completes.
  output:
    links:
      - title: Pull Request
        url: ${{ steps["create-pull-request"].output.remoteUrl }}
        icon: github
```

| Enfoque | Velocidad | Calidad de Metadatos | Esfuerzo del Desarrollador | Precisión |
| :--- | :--- | :--- | :--- | :--- |
| **Descubrimiento Automatizado** | Rápido (importación masiva) | Más baja (inferida) | Ninguno | Se degrada con el tiempo |
| **Registro Manual** | Lento (uno por uno) | Más alta (curada) | Alto | Depende de la rigurosidad |
| **Híbrido (PRs con Scaffolder)** | Medio | Media-Alta | Bajo (revisión de PR) | Mantenida mediante revisión de código |

> **Tabla 6.4** - Comparativa de estrategias de descubrimiento e incorporación al catálogo

---

### Comparativa: Backstage OSS frente a Soluciones Comerciales (COTS)

| Factor | Backstage OSS | Spotify Portal | Port | OpsLevel | Cortex | Harness IDP | Compass |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Tiempo hasta el primer valor** | 2-4 meses | 4-6 semanas | 2-4 semanas | 2-4 semanas | 2-4 semanas | 2-4 semanas | 2-4 semanas |
| **Profundidad de personalización** | Ilimitada | Alta | Media | Media | Media | Media | Baja |
| **Mantenimiento continuo** | Alto (1+ FTE) | Medio | Bajo | Bajo | Bajo | Bajo | Bajo |
| **Ecosistema de plugins** | 200+ Comunidad | 200+ Gestionados | Curados | Curados | Curados | Nativo + Comunidad | Atlassian |
| **Punto fuerte principal** | Flexibilidad | Backstage gestionado | Flujos de trabajo | Puntuación de madurez | Gobernanza | Integración CI/CD | Ecosistema Atlassian |
| **Ideal para** | Grandes organizaciones, necesidades a medida | Backstage sin sobrecarga de mantenimiento | Rápido retorno de inversión | Foco en SRE / operaciones | Alto nivel de cumplimiento | Usuarios de la pila Harness | Entornos Atlassian |

> **Tabla 6.5** - Comparativa de soluciones de portales de desarrolladores

> **Figura 6.4** - Vista general de la herramienta de evaluación Portal-IQ

> **Figura 6.5** - Matriz de comparación de características generada por Portal IQ

#### Lista de Verificación para Puesta en Producción del Portal
- [ ] El portal solo muestra capacidades reales y operativas (sin paneles vacíos ni marcadores de posición).
- [ ] Cada servicio del catálogo cuenta con un propietario explícito mapeado a una entidad `Group` de Backstage.
- [ ] Los desarrolladores pueden descubrir cualquier servicio, propietario y documentación sin recurrir a canales informales o wikis.
- [ ] El SSO está activo y el modo invitado deshabilitado.
- [ ] Al menos un flujo de alto valor (como la incorporación de servicios con Scaffolder) está totalmente automatizado.
- [ ] TechDocs contiene documentación para los servicios principales del equipo de plataforma.
- [ ] El uso del portal (visitas a páginas y búsquedas en el catálogo) se monitoriza activamente.

---

### Resumen

- **Los portales son interfaces, no plataformas**: Un portal desplegado antes de contar con las capacidades de backend crea un "portal del peligro".
- **Evaluación estructurada**: Analiza opciones a través de los seis planos de la plataforma (Catálogo, Golden Paths, Operaciones, Seguridad, Observabilidad y SEI).
- **Enfoque MVP**: Habilita inicialmente Catálogo, TechDocs y Búsqueda; posterga cuadros de mando y copilotos de IA hasta alcanzar la madurez necesaria.
- **SSO obligatorio**: Integra Keycloak antes de abrir el portal a los usuarios.
- **Estructura organizativa previa**: Define usuarios y grupos en el catálogo para que la propiedad sea explícita y verificable.
- **Registro mediante GitOps y Scaffolder**: Automatiza la incorporación de servicios mediante solicitudes de extracción (PR) estandarizadas.
- **Equilibrio entre velocidad y precisión**: Combina el descubrimiento automatizado con la curación asistida por plantillas.

---

### Referencias

- **[1]** *Backstage Open Source Developer Portal Framework*. [https://backstage.io](https://backstage.io/)
- **[2]** *Backstage Composability API Documentation*. [https://backstage.io/docs/plugins/composability/](https://backstage.io/docs/plugins/composability/)
- **[3]** *Backstage Architecture Overview*. [https://backstage.io/docs/overview/architecture-overview/](https://backstage.io/docs/overview/architecture-overview/)
- **[4]** *Portal IQ — Developer Portal Evaluation and Recommendation Tool*. [https://portal-iq.platformetrics.com](https://portal-iq.platformetrics.com/)
