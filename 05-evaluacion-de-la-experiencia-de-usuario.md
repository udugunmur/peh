# Parte 1: Diseño, Construcción y Despliegue de la Plataforma de Ingeniería Principal

## Capítulo 5: Evaluación de la Experiencia de Usuario

Con el núcleo de una plataforma MVP ya establecido, el equipo de María en NewTech comenzará a evaluar la experiencia que tendrán sus desarrolladores al desplegar una aplicación de demostración a través de la plataforma. El propósito de este enfoque es establecer una **línea base de experiencia** (*experience baseline*) con la cual María pueda comparar a medida que la plataforma madure y el equipo recree y redespliegue la aplicación.

La ingeniería de plataformas transforma de raíz cómo los desarrolladores interactúan con la infraestructura mediante la creación de capas de autoservicio hoy —y, previsiblemente, capas autoevolutivas en el futuro— que ayudan a los desarrolladores a navegar por sistemas complejos. Esto elimina la principal fuente de fricción, reduciendo significativamente su sobrecarga cognitiva. Los desarrolladores rara vez desean, o tienen tiempo para, convertirse en expertos en infraestructura (lo que hoy en día suele ser un eufemismo para los servicios en la nube). ¿Cuál es entonces la alternativa?

---

### La Experiencia del Desarrollador (DevEx) como Columna Vertebral de las Plataformas

Desde el surgimiento del movimiento formal de DevOps, e incluso mucho antes, se había generado inadvertidamente una dependencia de los desarrolladores respecto a un grupo especializado denominado ingenieros DevOps, ingenieros de sistemas o títulos afines. El objetivo original era claro: ¿cómo conseguir que los desarrolladores se concentren en lo esencial —construir productos que generen ingresos— creando una estructura de soporte a su alrededor? Teóricamente tenía sentido, hasta que dejó de tenerlo. 

Cuando 100 desarrolladores dependen de cinco especialistas en infraestructura que a menudo carecen de habilidades de desarrollo de software, surgen múltiples problemas:
- **Cuellos de botella de ancho de banda** para atender las peticiones.
- **Conflictos de priorización** para determinar qué desarrollador o proyecto es más urgente.
- **Problemas de calidad** cuando el especialista en infraestructura realiza cambios sin comprender el objetivo final del software.
- **Sobrecarga de requisitos no funcionales (NFRs)** que los desarrolladores deben integrar manualmente para lograr un despliegue en producción sin fricciones.

La respuesta a estos desafíos fue situar la **Experiencia del Desarrollador (*Developer Experience* - DevEx)** como métrica fundamental de la ingeniería de plataformas. La evolución se orientó a proporcionar entornos preconfigurados, despliegues basados en plantillas y cumplimiento automatizado de seguridad y gobernanza. La ingeniería de plataformas abordó esto descomponiendo la complejidad en componentes reutilizables y mecanismos de autoservicio sin necesidad de asistencia manual constante.

Con el tiempo, se superó el enfoque puramente centrado en herramientas (preguntas como *¿Acaso la ingeniería de plataformas no es solo CI/CD?* o *¿Por qué la ingeniería de plataformas es DevOps hecho por desarrolladores?*). El factor diferenciador radica en que las capacidades de la plataforma se conciben bajo un ciclo de vida formal de producto: planificación, diseño, construcción, despliegue y recopilación continua de retroalimentación iterativa mediante MVPs.

> **Figura 5.1** - Principios clave de diseño para reducir la fricción del desarrollador

Las interfaces de la plataforma se diseñan teniendo en mente el flujo de trabajo del desarrollador, convirtiéndose en sistemas autodocumentados con abstracciones ajustadas a cada perfil (*persona*) y aplicando el principio de que prevenir errores en tiempo de diseño es mucho más eficaz que corregirlos en producción.

A medida que DevEx se convirtió en el motor de la ingeniería de plataformas, su medición cobró protagonismo [1], vinculándose con la reducción de las métricas **DORA** alineadas con los resultados de negocio. Los tres pilares fundamentales de DevEx son:
1. **Eficiencia**: Reducción del tiempo dedicado a tareas repetitivas.
2. **Satisfacción**: Disminución de la fricción para los usuarios finales (desarrolladores).
3. **Impacto**: Aporte directo al valor del negocio (generación de ingresos o prevención de pérdidas).

> **Figura 5.2** - Tres ejes de los KPIs de la plataforma

El enfoque de producto asegura la componibilidad y reemplazabilidad de las soluciones, evitando el bloqueo de proveedores (*vendor lock-in*).

---

### Despliegue desde la Perspectiva del Usuario

En esta sección simularemos la experiencia completa del desarrollador desde el cambio en el código hasta la aplicación en ejecución, identificando brechas entre las capacidades de la plataforma y las necesidades reales del desarrollador.

#### Definición de la Aplicación de Demostración

La aplicación de demostración es una API en **Node.js Express** que simula un despliegue típico de microservicios con múltiples endpoints para gestión de usuarios y procesamiento de pedidos. Incorpora rastreo distribuido (*distributed tracing*) realizando llamadas a servicios externos (como una pasarela de pago), consultas a bases de datos con creación de tramos (*spans*) anidados y emisión de métricas de negocio personalizadas (usuarios activos y duración de peticiones). La aplicación está instrumentada con **OpenTelemetry** para capturar automáticamente peticiones HTTP, llamadas a base de datos y eventos de negocio, enviando la telemetría a los recolectores de la plataforma.

#### Paso 1: Flujo de Trabajo en CI/CD

El desarrollador realiza una confirmación de código (*push*), activando la canalización automatizada sin intervención manual:

```yaml
# .github/workflows/deploy.yml
name: Complete Developer Deployment Experience
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # 1. Code checkout
      - uses: actions/checkout@v3
      # 2. Automated testing
      - name: Run Tests
        run: |
          npm install
          npm test
          npm run lint
      # 3. Build application
      - name: Build Application
        run: |
          npm run build
          echo "Build completed at $(date)"
      # 4. Container creation
      - name: Build and Push Container
        env:
          REGISTRY: ghcr.io
          IMAGE_NAME: ${{ github.repository }}
        run: |
          docker build -t $REGISTRY/$IMAGE_NAME:${{ github.sha }} .
          docker push $REGISTRY/$IMAGE_NAME:${{ github.sha }}
      # 5. Deploy to environment
      - name: Deploy to Platform
        run: |
          kubectl set image deployment/myapp \
            myapp=$REGISTRY/$IMAGE_NAME:${{ github.sha }} \
            --namespace=${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
```

#### Paso 2: Contenedorización Multi-etapa

Construcción optimizada de la imagen Docker ejecutada con usuario sin privilegios (*non-root*):

```dockerfile
# Multi-stage Dockerfile for Node.js application
# Stage 1: Build stage
FROM node:24-alpine AS builder
WORKDIR /app
# Cache dependencies
COPY package*.json ./
RUN npm ci --only=production
# Copy source and build
COPY . .
RUN npm run build

# Stage 2: Production stage
FROM node:24-alpine
# Security: Run as non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001
WORKDIR /app
# Copy only necessary files from builder
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./
# Health check for container orchestration
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js
USER nodejs
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

#### Paso 3: Conciliación Continua con GitOps (Argo CD)

Declaración de la aplicación en Argo CD para sincronización y autoreparación continua [3]:

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: developer-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/app-deployments
    targetRevision: HEAD
    path: environments/production
    # Helm values or Kustomize
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

#### Paso 4: CLI de Autoservicio para Despliegues

Interfaz por línea de comandos para facilitar el despliegue a los desarrolladores sin exigir conocimientos profundos de infraestructura:

```bash
#!/bin/bash
# platform-deploy.sh - Self-service deployment script

# Developer-friendly deployment command
deploy_app() {
    local APP_NAME=$1
    local ENVIRONMENT=$2
    echo "Deploying $APP_NAME to $ENVIRONMENT environment..."
    
    # Validate application configuration
    if ! validate_config "$APP_NAME"; then
        echo "Configuration validation failed"
        exit 1
    fi
    
    # Auto-generate Kubernetes manifests
    generate_manifests "$APP_NAME" "$ENVIRONMENT"
    
    # Apply security policies
    apply_security_policies "$APP_NAME"
    
    # Deploy with automatic rollback on failure
    kubectl apply -f "./deployments/$APP_NAME/" \
        --namespace="$ENVIRONMENT" \
        --wait --timeout=300s || rollback "$APP_NAME" "$ENVIRONMENT"
        
    # Set up monitoring and alerts
    configure_monitoring "$APP_NAME"
    
    # Provide access endpoints
    echo "Deployment successful!"
    echo "Access your application at: https://$APP_NAME.$ENVIRONMENT.platform.company.com"
    echo "Monitoring dashboard: https://monitoring.company.com/dashboard/$APP_NAME"
}

# Simple developer interface
# Usage: ./platform-deploy.sh myapp production
deploy_app "$@"
```

#### Paso 5: Despliegue Seguro Reforzado

Manifiesto de Kubernetes con directivas estrictas de seguridad (AppArmor, contexto no root, sistema de archivos de solo lectura y límites de recursos):

```yaml
# secure-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  labels:
    app.kubernetes.io/name: demo-app
    app.kubernetes.io/managed-by: platform-deploy
  annotations:
    # Security scanning annotations
    security.scan/enabled: "true"
    security.scan/severity: "CRITICAL,HIGH"
spec:
  replicas: 3
  template:
    metadata:
      annotations:
        # Force security policies
        container.apparmor.security.beta.kubernetes.io/app: runtime/default
    spec:
      # Security context for pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: registry.company.com/apps/demo-app:1.0.0
          # Container security context
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          # Resource limits prevent resource exhaustion
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          # Liveness and readiness for resilience
          livenessProbe:
            httpGet:
              path: /health
              port: http # named port — avoid hardcoded numbers
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
              scheme: HTTPS
            initialDelaySeconds: 5
            periodSeconds: 5
```

> **Ejercicio 5.1**
> 
> Desarrolla un script de remediación automatizada de seguridad basado en el código de despliegue anterior para:
> 1. Corregir cualquier acceso exclusivo por HTTP forzando redirección HTTPS.
> 2. Añadir políticas de red ausentes para aislar el tráfico entre namespaces.

#### Despliegue Sin Fricción (*Zero-Friction Deployment*)

El ideal de la ingeniería de plataformas se alcanza cuando el desarrollador solo proporciona el nombre de la aplicación y la URL del repositorio Git, y la plataforma gestiona automáticamente compilaciones, TLS, escalado, monitorización, respaldos y políticas [2]:

```bash
#!/bin/bash
# frictionless-deploy.sh - Complete zero-friction deployment

deploy_with_everything() {
    local APP_NAME=$1
    local GIT_REPO=$2

    cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: $APP_NAME
---
apiVersion: platform.company.com/v1
kind: Application
metadata:
  name: $APP_NAME
  namespace: $APP_NAME
spec:
  source:
    git: $GIT_REPO
    branch: main
  build:
    type: automatic
    dockerfile: Dockerfile
  deployment:
    replicas: 3
    strategy: RollingUpdate
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPU: 70
  networking:
    ingress:
      enabled: true
      hostname: $APP_NAME.platform.company.com
      tls:
        enabled: true
        issuer: letsencrypt-prod
  monitoring:
    enabled: true
    alerts:
      - type: availability
        threshold: 99.9
      - type: latency
        threshold: 500ms
      - type: error-rate
        threshold: 1
  backup:
    enabled: true
    schedule: "0 2 * * *"
    retention: 30
  security:
    scanning:
      enabled: true
      schedule: "0 */6 * * *"
    networkPolicy:
      enabled: true
      type: strict
    secrets:
      autoRotation: true
      rotationDays: 90
EOF

    echo " Application deployed with:"
    echo " • Automatic builds from Git"
    echo " • TLS certificates (auto-renewed)"
    echo " • Autoscaling (2-5 replicas)"
    echo " • Monitoring & alerts"
    echo " • Daily backups (60-day retention)"
    echo " • Security scanning every 12 hours"
    echo " • Network policies enabled"
    echo " • Secret rotation every 90 days"
    echo ""
    echo " Access at: https://$APP_NAME.platform.company.com"
    echo " Dashboard: https://platform.company.com/apps/$APP_NAME"
}

# Usage: ./frictionless-deploy.sh my-app https://github.com/company/my-app
deploy_with_everything "$@"
```

---

### Entornos de Previsualización frente a Despliegue Local Completo

| Escenario | Entorno de Previsualización (*Preview*) | Entorno Local (*Local*) |
| :--- | :--- | :--- |
| **Depuración de un problema de producción** | **Ideal**: Refleja exactamente la infraestructura de producción, redes y dependencias reales sin riesgo de desconfigurar la máquina local. | Menos adecuado debido a la complejidad de replicar sistemas distribuidos completos. |
| **Desarrollo de una nueva funcionalidad** | Requiere esperar los tiempos de compilación y despliegue en cada cambio. | **Ideal**: Ciclo de retroalimentación instantáneo, recarga rápida (*hot reload*), ejecución paso a paso con *breakpoints* y acceso directo a base de datos. |
| **Revisión de una Pull Request (PR)** | **Ideal**: URLs efímeras generadas automáticamente permiten probar el comportamiento real de la aplicación en vivo sin descargar código localmente. | Obliga al revisor a cambiar de rama, instalar dependencias y levantar servicios localmente. |
| **Incorporación de un nuevo desarrollador (*Onboarding*)** | Útil para pruebas de integración posteriores una vez familiarizado. | **Ideal**: Permite explorar el código, experimentar y romper componentes de forma segura para construir modelos mentales del sistema. |

#### Directrices para el Equipo de Plataforma:
1. **Mantener paridad estricta**: Mismos nombres de variables de entorno y patrones de descubrimiento de servicios entre local y preview.
2. **Proveer datos de muestra**: Conjuntos de datos iniciales válidos para ambos contextos.
3. **Documentar el "cuándo" y no solo el "cómo"**: Guiar a los desarrolladores sobre cuándo utilizar cada entorno.
4. **Invertir en la experiencia local**: Un entorno local defectuoso genera retrabajo y fricción constante.
5. **Configurar TTLs en entornos de previsualización**: Destrucción automática de entornos efímeros para optimizar costes.

---

### Instrumentación de la Aplicación para Observabilidad

Siguiendo los principios del [Capítulo 4](https://subscription.packtpub.com/book/programming/9781806380138/4), la aplicación de demostración se instrumenta con OpenTelemetry en 4 pasos:

#### 1. Configuración de OpenTelemetry SDK (`instrumentation.js`)

```javascript
// instrumentation.js - OpenTelemetry Setup
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-proto');
const { OTLPMetricExporter } = require('@opentelemetry/exporter-metrics-otlp-proto');
const { PeriodicExportingMetricReader } = require('@opentelemetry/sdk-metrics');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_ENDPOINT || 'http://otel-collector.platform:4318/v1/traces',
  }),
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: 'http://otel-collector.platform:4318/v1/metrics',
    }),
    exportIntervalMillis: 10_000,
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

> **Nota**: `PeriodicExportingMetricReader` es indispensable; sin él, las métricas registradas no se exportarán a Grafana.

#### 2. Tramos Personalizados y Métricas de Negocio (`app.js`)

```javascript
// app.js - Custom Spans and Metrics
const { trace, metrics } = require('@opentelemetry/api');

const tracer = trace.getTracer('demo-app');
const meter = metrics.getMeter('demo-app');

// Custom metrics
const requestCounter = meter.createCounter('http_requests_total');
const requestDuration = meter.createHistogram('http_request_duration_ms');

app.get('/api/users/:id', async (req, res) => {
  const span = tracer.startSpan('fetch_user_data');
  const startTime = Date.now();
  
  try {
    span.setAttribute('user.id', req.params.id);
    
    // Nested span for database query
    const dbSpan = tracer.startSpan('database.query', {
      attributes: { 'db.operation': 'SELECT' }
    });
    const userData = await fetchUserFromDatabase(req.params.id);
    dbSpan.end();
    
    requestCounter.add(1, { endpoint: '/api/users' });
    requestDuration.record(Date.now() - startTime);
    
    res.json(userData);
  } catch (error) {
    span.recordException(error);
    throw error;
  } finally {
    span.end();
  }
});
```

#### 3. Registro Estructurado con Contexto de Trazas (`logger.js`)

Inyección automática de `traceId` y `spanId` en cada registro mediante Winston:

```javascript
// logger.js - Structured Logging with Trace Context
const winston = require('winston');
const { trace, context } = require('@opentelemetry/api');

const logger = winston.createLogger({
  format: winston.format.printf((info) => {
    const span = trace.getSpan(context.active());
    return JSON.stringify({
      timestamp: info.timestamp,
      level: info.level,
      message: info.message,
      traceId: span?.spanContext().traceId || 'no-trace',
      spanId: span?.spanContext().spanId || 'no-span',
    });
  }),
  transports: [new winston.transports.Console()],
});
```

#### 4. Configuración de Entorno en el Despliegue (`otel-deployment.yaml`)

```yaml
# otel-deployment.yaml - OTEL Environment Configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
spec:
  template:
    spec:
      containers:
        - name: app
          image: ghcr.io/company/demo-app:IMAGE_TAG_PLACEHOLDER
          env:
            - name: OTEL_SERVICE_NAME
              value: "demo-app"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.observability:4318"
            - name: NODE_OPTIONS
              value: "--require ./instrumentation.js"
```

---

### Exposición de una URL Pública

La plataforma abstrae la complejidad de balanceadores, DNS y certificados a especificaciones declarativas sencillas:

```yaml
annotations: {"kubernetes.io/ingress.class": "nginx"}
host: myapp.platform.company.com
```

Integración con **cert-manager** y el protocolo ACME para emisión y renovación automática de certificados TLS con **Let's Encrypt**:

```yaml
tls: {secretName: "myapp-tls", issuer: "letsencrypt-prod"}
annotations: {"cert-manager.io/cluster-issuer": "letsencrypt-prod"}
```

---

### Diarios de DevEx: Las Primeras Impresiones Importan

| Situación | Lección para la Plataforma | Sentimiento del Desarrollador |
| :--- | :--- | :--- |
| Confirmé código en `main`, la canalización se ejecutó y en 3 minutos el servicio estaba activo con TLS configurado automáticamente. Sin pelear con certificados ni configuraciones de NGINX. | Los valores por defecto seguros eliminan fricción inicial. Con TLS automático, los desarrolladores no adquieren malos hábitos de desplegar sin cifrado "solo para probar". | **Entusiasmo y asombro** |
| Al depurar una petición lenta, pulsé en "Ver registros" y el identificador de traza (*Trace ID*) exacto ya estaba vinculado sin tener que propagar manualmente identificadores en el código. | La observabilidad con configuración cero aporta valor inmediato al inyectar el contexto en la capa de infraestructura. | **Sorpresa y satisfacción** |
| El tráfico se multiplicó por 10 durante una demostración. El servicio escaló de forma fluida sin alertas porque el autoescalado venía activado por defecto. | Las configuraciones listas para producción generan confianza. Los desarrolladores se centran en las funcionalidades en lugar de la planificación de capacidad. | **Tranquilidad** |
| Mi pod fallaba continuamente con `Exit Code 137`. Pasé 2 horas investigando hasta que un compañero me indicó que significa `OOMKilled` (sin memoria). | Los mensajes de error son documentación en el punto de necesidad. Traducir códigos de salida de Kubernetes a explicaciones legibles ahorra tiempo valioso. | **Fricción y frustración** |
| Desplegué un error en producción. Encontré el botón de "Revertir" en la interfaz: un clic y 30 segundos restauraron la versión previa sin necesidad de abrir un ticket de soporte. | La recuperación en autoservicio transforma los errores en oportunidades de aprendizaje y fomenta despliegues más frecuentes y seguros. | **Empoderamiento** |
| Creé un nuevo servicio usando la plantilla del equipo con CI/CD preconfigurado, sondas de salud, métricas y escaneo de seguridad. Fue como heredar 6 meses de mejores prácticas desde el primer día. | Los *Golden Paths* aceleran drásticamente la incorporación de desarrolladores y evitan reinventar patrones básicos. | **Satisfacción** |

---

### Resumen

- Despliega una **aplicación de demostración** para evaluar la experiencia de usuario de la plataforma y utilizarla como base en la incorporación de nuevos desarrolladores.
- Las plataformas deben proporcionar **autoservicio real** que equilibre velocidad y gobernanza, eliminando traspasos manuales y tickets de aprobación.
- Las confirmaciones de código deben desencadenar automáticamente pruebas, contenedorización, publicación en registros y sincronización con GitOps.
- Integra **escaneo de seguridad**, políticas de red y validación de RBAC desde el inicio de la canalización.
- Utiliza **OpenTelemetry** para instrumentación neutral y tramos personalizados para lógica de negocio.
- Inyecta el **contexto de traza en los registros estructurados** para correlación inmediata.
- Haz que **HTTPS sea el camino de menor resistencia** mediante gestión automatizada de certificados con cert-manager y Let's Encrypt.
- Oculta la complejidad de enrutamiento e Ingress mediante especificaciones declarativas sencillas.
- Proporciona soporte equilibrado tanto para **entornos locales** como para **entornos de previsualización (*preview*)**.
- Mide el éxito de la plataforma mediante los tres ejes de DevEx: **eficiencia, satisfacción e impacto**.

---

### Referencias

- **[1]** *Atlassian Developer Experience Framework*. [https://www.atlassian.com/developer-experience](https://www.atlassian.com/developer-experience)
- **[2]** *Developer Experience Book — Principles of Frictionless Software Delivery*. [https://developerexperiencebook.com/](https://developerexperiencebook.com/)
- **[3]** *Argo CD Reconcile Architecture and Operation Manual*. [https://argo-cd.readthedocs.io/en/stable/operator-manual/reconcile/](https://argo-cd.readthedocs.io/en/stable/operator-manual/reconcile/)
