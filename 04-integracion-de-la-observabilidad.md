# Parte 1: Diseño, Construcción y Despliegue de la Plataforma de Ingeniería Principal

## Capítulo 4: Integración de la Observabilidad

A medida que las organizaciones de ingeniería se esfuerzan por construir plataformas más autónomas para reducir los traspasos (*handoffs*), la sobrecarga cognitiva y la fricción de los desarrolladores al crear sus productos, la **observabilidad** se consolida como la piedra angular de dicha autonomía. Las buenas prácticas de observabilidad mejoran la estabilidad general del sistema y permiten a los equipos diagnosticar y solucionar problemas con rapidez. 

El físico del siglo XIX Lord Kelvin dijo una vez: *"Lo que no se puede medir, no se puede mejorar"*. Más tarde, Peter Drucker aplicó esto al mundo empresarial moderno afirmando que no se puede gestionar lo que no se puede medir. Cuando se trata de plataformas de ingeniería, esto resulta indiscutible. Sin embargo, la cuestión clave radica en qué medir, cómo medirlo y cómo comunicar lo que se mide. Las respuestas a todas estas preguntas fundamentales residen en la observabilidad.

El objetivo de las plataformas es reducir la sobrecarga cognitiva. El ciclo de retroalimentación necesario para la autoreparación (*self-healing*) —y hoy en día, con el advenimiento de la IA agéntica, la autoevolución (*self-evolving*)— requiere información procesable y significativa que pueda medirse y mejorarse sin excesiva intervención manual y alineada con los estándares de la industria.

---

### Diferenciación frente a la Monitorización

La monitorización y la observabilidad no son lo mismo, aunque son disciplinas estrechamente relacionadas:
- **Monitorización**: Es la práctica de comprobar si un sistema está operativo. Si el sistema se cae, la monitorización lo detecta y se soluciona manualmente o por medios automatizados. Aborda los **"conocidos conocidos"** (*known knowns*); por ejemplo, un proceso falla porque se quedó sin espacio en disco y el programa no estaba preparado para gestionarlo.
- **Observabilidad**: Es la capacidad de inferir los estados internos de un sistema examinando sus salidas externas (métricas, registros y trazas). Se centra en los **"desconocidos desconocidos"** (*unknown unknowns*), situaciones imprevistas donde el sistema se comporta de forma inesperada. Por ejemplo, al desplegar una nueva versión, ciertas peticiones de usuarios comienzan a fallar debido a una condición de carrera entre microservicios generada por código raramente utilizado que nunca se había probado en conjunto.

---

### La Observabilidad como Habilitador del Negocio

Los datos recopilados como parte de la iniciativa de observabilidad son un producto en sí mismos, capaces de dar servicio a diversos interesados (*stakeholders*): departamentos de finanzas, ventas, marketing, gestión de producto y directivos. Sin embargo, cada perfil necesita interpretaciones distintas de los mismos datos.

En lugar de construir paneles de datos aislados para cada departamento, el enfoque óptimo es un **Panel Único de Control** (*Single Pane of Glass* - **SPOG**). En entornos empresariales como NewTech, donde ya existen múltiples herramientas contratadas (monitorización de red, infraestructura, aplicaciones, cumplimiento, etc.), el SPOG no busca reemplazar dichas herramientas, sino integrar programáticamente sus datos (incluso mediante servidores MCP proporcionados por ellas) para ofrecer una vista de alto nivel del flujo global del sistema.

Esto evita la aparición de "TI en la sombra" (*shadow IT*) al proporcionar paneles de autoservicio para métricas de negocio y demuestra el valor intrínseco y el ROI de la inversión en la plataforma.

> **Figura 4.1** - Datos de observabilidad descritos a través de múltiples facetas de operaciones de ingeniería y de negocio. Reproducido con permiso de *Effective Platform Engineering*, Manning, 2025, Chankramath et.al

---

### Desarrollo Guiado por la Observabilidad (ODD)

El **Desarrollo Guiado por la Observabilidad** (*Observability-Driven Development* - **ODD**) es una disciplina que convierte la observabilidad en una prioridad de primer nivel para los desarrolladores en lugar de una ocurrencia tardía, de forma análoga a TDD (*Test-Driven Development*) o BDD (*Behavior-Driven Development*). 

Los equipos de plataforma proporcionan estas capacidades como requisitos no funcionales (*Non-Functional Requirements* - NFRs) para facilitar su adopción. ODD amplía los conceptos de TDD centrándose en los datos de telemetría necesarios para comprender el comportamiento del software en producción bajo condiciones reales.

---

### Recopilación de Métricas, Logs y Trazas

#### Definición de los Tres Pilares de la Observabilidad

1. **Métricas**: Indican *qué sucedió*. Se presentan como series temporales agregadas a lo largo de intervalos regulares (no por evento individual).
2. **Logs (Registros)**: Detallan *cómo sucedió*. Representan eventos discretos estructurados, indispensables para el análisis forense de incidentes.
3. **Trazas (Traces)**: Explican *por qué sucedió*. Proporcionan correlación de extremo a extremo para cada petición a través de múltiples servicios distribuidos.

La combinación de estos tres pilares permite reconstruir el estado interno del sistema. El rastreo distribuido (*tracing*), a menudo infrautilizado, es crítico porque en arquitecturas de microservicios el síntoma visible de un fallo rara vez se localiza en el componente que originó la causa raíz.

> **Figura 4.2** - Vista estandarizada de los componentes de la observabilidad

#### Base de Datos de Series Temporales y su Relevancia

Una base de datos de métricas de series temporales como **Prometheus** está diseñada específicamente para entornos dinámicos nativos de la nube. Permite recolectar métricas a intervalos periódicos, almacenarlas con marcas de tiempo e indexarlas para consultas eficientes mediante **PromQL**.

Mientras que los registros indican qué falló y las trazas señalan dónde se produjo la lentitud, las métricas permiten detectar que el sistema se está degradando **antes** de que falle, habilitando capacidades como el autoescalado basado en datos.

#### Estandarización con OpenTelemetry

**OpenTelemetry (OTEL)** [1] es un marco de trabajo de código abierto compuesto por APIs, SDKs y herramientas para recolectar datos de telemetría de aplicaciones e infraestructura bajo un formato neutral e independiente del proveedor mediante el protocolo **OTLP** (*OpenTelemetry Protocol*).

> **Figura 4.3** - Componentes de OpenTelemetry

Beneficios de la estandarización con OTEL:
- Proporciona una única API para métricas, registros y trazas en todos los servicios.
- Permite cambiar de proveedor de backend sin modificar el código de la aplicación.
- La instrumentación se realiza una sola vez.
- Aplica convenciones semánticas uniformes en nombres de atributos y etiquetas entre equipos.
- Facilita la unificación de múltiples herramientas heterogéneas en grandes empresas.

Ejemplo de instrumentación de una aplicación con OpenTelemetry:

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

# Initialize OpenTelemetry
resource = Resource(attributes={
    "service.name": "payment-service",
    "deployment.environment": "production"
})
provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(OTLPSpanExporter(
    endpoint="otel-collector:4317" # Your collector endpoint standard
))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

# Get a tracer
tracer = trace.get_tracer(__name__)

# Instrument your code
def process_payment(user_id, amount):
    with tracer.start_as_current_span("process_payment") as span:
        # Add custom attributes
        span.set_attribute("user.id", user_id)
        span.set_attribute("payment.amount", amount)
        span.set_attribute("payment.currency", "USD")
        
        # Your business logic
        validate_user(user_id)
        charge_card(amount)
        
        span.set_attribute("payment.status", "success")
        return {"status": "completed"}

def validate_user(user_id):
    with tracer.start_as_current_span("validate_user"):
        # Validation logic
        return True

def charge_card(amount):
    with tracer.start_as_current_span("charge_card") as span:
        span.set_attribute("card.processor", "stripe")
        # Payment processing
        return True
```

> **Fragmento de Código 4.1**: Instrumentación de aplicación con OpenTelemetry

#### Arquitectura de Panel Único de Control (SPOG)

El SPOG interconecta los tres pilares mediante identificadores comunes (como `trace_id`), permitiendo navegar de una traza lenta a sus registros y métricas asociados.

Ventajas clave de un SPOG:
- **Interfaz de consulta centralizada** [5]: Consulta métricas de Prometheus, logs de Loki/Elasticsearch y trazas de Tempo/Jaeger desde una interfaz unificada (como Grafana).
- **Análisis de causa raíz (RCA) simplificado**: Correlación visual entre tipos de telemetría.
- **Mapas de servicios y gráficos de dependencias**: Representación homogénea de la topología del sistema.
- **Reducción del tiempo medio de resolución (MTTR)**: Disminuye la sobrecarga y el cambio de contexto (*context switching*) durante la resolución de incidencias.

---

### Ingesta Automática de Telemetría

#### Responsabilidad de la Plataforma

En el modelo de responsabilidad compartida:
- **Equipo de Plataforma**: Mantiene los recolectores de OpenTelemetry (*OTEL Collectors*) desplegados en los nodos (habitualmente como un `DaemonSet`), gestiona el raspado (*scraping*), descubrimiento de servicios, almacenamiento en buffer, reintentos y escalado de las bases de datos de series temporales con políticas de retención adecuadas.
- **Equipos de Desarrollo**: Instrumentan el código de sus aplicaciones utilizando los SDKs de OTEL y plantillas proporcionadas por la plataforma.

#### Métodos de Ingesta (Push frente a Pull)

- **Método Pull (Métricas)**: Ideal para métricas. Servidores como Prometheus consultan periódicamente los endpoints `/metrics` expuestos por los pods mediante descubrimiento de servicios de Kubernetes (`ServiceMonitor` CRDs). Evita la necesidad de buffers pesados en el cliente.

```python
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

@app.route('/metrics')
def metrics():
    # Prometheus scrapes this endpoint
    return generate_latest()

@app.route('/api/payment')
def payment():
    http_requests_total.labels(
        method='GET',
        endpoint='/api/payment',
        status='200'
    ).inc()
    return {"status": "ok"}
```

> **Fragmento de Código 4.2**: Demostración del modo de ingesta Pull

- **Método Push (Trazas)**: Ideal para trazas. Las aplicaciones envían los tramos (*spans*) directamente al endpoint del recolector OTLP en el puerto `4317` tan pronto como se completan las operaciones.

```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

exporter = OTLPSpanExporter(endpoint="otel-collector:4317")
tracer = trace.get_tracer(__name__)

@app.route('/api/payment')
def payment():
    with tracer.start_as_current_span("payment") as span:
        span.set_attribute("amount", 100)
        # Span automatically pushed to collector on exit
        return {"status": "ok"}
```

> **Fragmento de Código 4.3**: Demostración del modo de ingesta Push

- **Método Híbrido (Logs y Aplicaciones Integradas)**: Combina recolección Pull para métricas y emisión Push para trazas y logs estructurados hacia plataformas como Loki.

```python
from prometheus_client import Counter, generate_latest
from opentelemetry import trace

# Pull: Prometheus scrapes /metrics
request_count = Counter('requests_total', 'Total requests')

# Push: OTLP sends traces immediately
tracer = trace.get_tracer(__name__)

@app.route('/metrics')
def metrics():
    return generate_latest() # Pull endpoint

@app.route('/api/payment')
def payment():
    request_count.inc() # Increments counter (pulled later)
    with tracer.start_as_current_span("payment"): # Pushes span now
        return {"status": "ok"}
```

> **Fragmento de Código 4.4**: Demostración del modo de ingesta híbrido

#### El Dilema entre Construir o Comprar (*Build vs. Buy Tradeoff*)

| Nivel de Madurez | Perfil de la Organización | Enfoque Recomendado | Justificación | Implementación Típica |
| :--- | :--- | :--- | :--- | :--- |
| **Startup / Fase Temprana** (1-50 servicios) | Equipo pequeño (1-2 ingenieros de plataforma), presupuesto limitado, necesidad de iteración rápida | **Comprar** (SaaS comercial) | Salida al mercado crítica; sin capacidad de mantenimiento de infraestructura; 2K$-5K$/mes es más barato que contratar personal | Plataforma todo-en-uno como Datadog o New Relic |
| **Fase de Crecimiento** (50-200 servicios) | Equipo en expansión (3-5 ingenieros de plataforma), presupuesto moderado, foco en estandarización | **Híbrido** (Código abierto + Compra selectiva) | Construir métricas/trazas principales con herramientas CNCF; comprar gestión de logs y respuesta a incidentes donde la carga operativa es mayor | Prometheus + Grafana + Tempo (construir) + PagerDuty + Elasticsearch (comprar) |
| **Escala Empresarial** (200-1000+ servicios) | Equipo dedicado de observabilidad (5-10 ingenieros), presupuesto significativo y requisitos de cumplimiento | **Construir** (Código abierto con extensiones) | La economía de escala favorece la construcción; soberanía de datos e integraciones heredadas necesarias | Thanos + Grafana + Tempo + Loki + exportadores personalizados |
| **Plataforma como Producto** (La observabilidad es el negocio) | Equipo de observabilidad especializado (15+ ingenieros), la observabilidad diferencia el producto | **Construir** (Pila completamente personalizada) | Propiedad intelectual central; ventaja competitiva con capacidades exclusivas; aislamiento de datos de clientes | TSDB personalizada, lenguaje de consulta propietario, arquitectura multi-tenant |
| **Industrias Reguladas** (Finanzas, Salud) | Cualquier tamaño, estrictos requisitos de residencia de datos y auditoría | **Construir** (*On-premises* / Nube privada) | Mandatos de cumplimiento impiden SaaS; las pistas de auditoría deben permanecer dentro de los límites de infraestructura | Prometheus/Grafana autohospedado en entornos aislados (*air-gapped*) |

> **Tabla 4.1** - Matriz de decisión de Construir frente a Comprar según la madurez organizacional

---

### Perfiles de Observabilidad (*Observability Personas*)

#### Necesidades Diversas de los Interesados

> **Figura 4.4** - Perfiles típicos para el consumo de datos de observabilidad y su propósito principal

- **Usuarios finales externos**: Requieren información clara sobre el estado de los servicios (páginas de estado y cumplimiento de SLA).
- **Dirección ejecutiva**: Necesita KPIs de negocio (tasas de conversión, tiempos de entrega de características e impacto en ingresos).
- **Desarrolladores de aplicaciones**: Precisan depuración a nivel de traza, registros de errores y disponibilidad de APIs internas.
- **Gestores de producto (Product Managers)**: Analizan patrones de uso de funcionalidades y puntos de fricción del usuario.
- **Ingenieros de control de calidad (QA)**: Validan paridad entre versiones y rendimiento de suites de pruebas.
- **Equipos de DevOps y SRE**: Monitorean incidentes en tiempo real, consumo de presupuestos de error y estabilidad del clúster.
- **Equipos de Gobernanza y Cumplimiento**: Validan registros de auditoría de acceso a datos sensibles (PII).
- **Seguridad**: Investigan trazas de autenticación, abusos y rutas de ataque.

> **Figura 4.5** - Ejemplo de comunicación de incidentes (caso Cloudflare)

#### Paneles Personalizados por Roles

> **Figura 4.6** - Proceso de 10 pasos para construir plataformas de observabilidad de clase mundial

Recomendaciones para la gestión de paneles en **Grafana**:
1. Proveer paneles preconfigurados como código almacenados en Git.
2. Utilizar plantillas con variables dinámicas por entorno y servicio.
3. Superponer marcadores de despliegue y ventanas de mantenimiento sobre las gráficas temporales.
4. Restringir permisos de edición manteniendo acceso de solo lectura para la mayoría de usuarios.
5. Optimizar consultas mediante el inspector de Grafana para evitar sobrecargar los orígenes de datos.

#### Observabilidad de Seguridad

Integrar la seguridad en la observabilidad implica exponer violaciones de políticas de OPA, registros de auditoría y recuentos de vulnerabilidades de imágenes como métricas estándar.

Ejemplo de integración de escáner de vulnerabilidades (Trivy) con Prometheus:

```python
import subprocess
import json
from prometheus_client import Gauge, generate_latest

# Metrics for vulnerability tracking
vulnerability_gauge = Gauge(
    'image_vulnerabilities_by_severity',
    'Container image vulnerabilities',
    ['image', 'severity']
)

def scan_image_and_export_metrics(image_name):
    # Run Trivy scanner [3]
    result = subprocess.run(
        ['trivy', 'image', '--format', 'json', image_name],
        capture_output=True,
        text=True
    )
    scan_data = json.loads(result.stdout)
    
    # Count vulnerabilities by severity
    severity_counts = {'CRITICAL': 0, 'HIGH': 0, 'MEDIUM': 0, 'LOW': 0}
    for target in scan_data.get('Results', []): # iterate all targets
        for vuln in target.get('Vulnerabilities', []): # OS pkgs, language deps, etc.
            severity = vuln.get('Severity')
            if severity in severity_counts:
                severity_counts[severity] += 1

    # Export to Prometheus (sanitized - no CVE details)
    for severity, count in severity_counts.items():
        vulnerability_gauge.labels(
            image=image_name,
            severity=severity
        ).set(count)

# Schedule periodic scans
scan_image_and_export_metrics('payment-service:v1.2.3')
```

> **Fragmento de Código 4.5**: Integración de escáner CVE con Prometheus

> **Figura 4.7** - Ejemplo de panel de visualización de vulnerabilidades CVE

---

### Integración de la Observabilidad en CI/CD

#### La Observabilidad en la Canalización de CI/CD

- **Validación obligatoria de instrumentación**: La canalización de CI debe fallar si los microservicios no exponen endpoints de métricas, registros estructurados o tramos de trazas, impidiendo que llegue a producción código sin instrumentar.
- **Telemetría de la propia canalización**: Monitoreo de tiempos de compilación, duración de pruebas y frecuencia de despliegue para optimizar cuellos de botella del flujo de entrega.

#### Alertas Procesables y Reducción de Ruido

Reglas para el diseño de alertas efectivas:
- Alertar sobre **síntomas** que afecten a los usuarios (tasas de error HTTP, violaciones de latencia P95/P99), no sobre causas internas (uso transitorio de CPU).
- Cada alerta debe responder explícitamente a: *¿Qué acción inmediata debo tomar?* e incluir enlaces directos a manuales operativos (*runbooks*).
- Implementar alertas basadas en la tasa de consumo de presupuestos de error (*burn rate*): consumo rápido (*fast burn*) para caídas graves y consumo lento (*slow burn*) para degradaciones graduales.
- Silenciar alertas durante ventanas de mantenimiento conocidas y podar periódicamente alertas que no requirieron intervención.

#### SLOs, SLIs y Confianza

- **SLI (Indicador de Nivel de Servicio)**: Métrica cuantificable que refleja la experiencia real del usuario (por ejemplo, ratio de peticiones exitosas frente a totales).
- **SLO (Objetivo de Nivel de Servicio)**: Umbral objetivo acordado para el SLI (por ejemplo, 99.9% de disponibilidad en un periodo móvil de 30 días).
- **Presupuesto de Error (Error Budget)**: El margen de fallos permitido (`100% - SLO`), utilizable para absorber el riesgo de nuevos despliegues e innovaciones.

> **Figura 4.8** - Ciclo de vida de despliegue guiado por la observabilidad

#### SLOs como Código

Definir los SLOs como manifiestos declarativos de Kubernetes almacenados en Git permite sincronizarlos mediante GitOps y utilizarlos en canalizaciones de CI/CD para detener promociones si se viola el presupuesto de error:

```yaml
apiVersion: monitoring.platform.io/v1
kind: ServiceLevelObjective
metadata:
  name: payment-service-availability
  namespace: production
spec:
  service: payment-service
  description: "Payment API availability SLO"
  # SLI Definition
  sli:
    type: availability
    query: |
      sum(rate(http_requests_total{job="payment-service",code!~"5.."}[5m])) / sum(rate(http_requests_total{job="payment-service"}[5m]))
  # SLO Target
  objective:
    target: 0.999 # 99.9% availability
    window: 30d # Monthly error budget
  # Alerting configuration
  alerting:
    burnRateAlerts:
      - severity: critical
        longWindow: 1h
        shortWindow: 5m
        burnRateFactor: 14.4 # Fast burn
      - severity: warning
        longWindow: 6h
        shortWindow: 30m
        burnRateFactor: 6 # Slow burn
  # Dashboard generation
  dashboard:
    enabled: true
    grafanaFolder: "SLO Dashboards"
```

> **Fragmento de Código 4.6**: Definición de SLOs como código

---

### Resumen

- La **monitorización** aborda los "conocidos conocidos" (caídas evidentes); la **observabilidad** desvela los "desconocidos desconocidos" (comportamientos emergentes y condiciones de carrera).
- El **Desarrollo Guiado por la Observabilidad (ODD)** exige que los servicios emitan métricas, registros estructurados y trazas antes de ser promovidos a producción.
- Los **tres pilares**: métricas (*qué pasó*), logs (*cómo pasó*) y trazas (*por qué pasó*).
- **OpenTelemetry (OTEL)** permite instrumentar el código una única vez y enviar datos vía OTLP (puerto 4317) a cualquier backend sin ataduras a proveedores.
- Las arquitecturas **SPOG** correlacionan trazas, métricas y registros mediante identificadores comunes en interfaces unificadas como Grafana.
- El modelo **Pull** es ideal para métricas (Prometheus); el modelo **Push** para trazas (OTLP) y el modelo **híbrido** para logs (Loki).
- Adapta los paneles a los distintos **perfiles de usuario** (desarrolladores, SREs, directivos, seguridad) a partir del mismo origen de datos.
- Define alertas basadas en síntomas y **tasas de consumo de presupuestos de error (*burn rate*)**, asociando cada alerta a un manual operativo.
- Modela los **SLOs como código** mediante recursos declarativos en Git para automatizar las decisiones de despliegue y reversión.

---

### Referencias

- **[1]** *OpenTelemetry Documentation — APIs, SDKs, OTLP Protocol Specification, Semantic Conventions, and Collector Configuration Reference*. [https://opentelemetry.io/docs/](https://opentelemetry.io/docs/)
- **[2]** *Grafana Documentation — Official Grafana Docs Covering Dashboard Creation, Data Source Integrations, and the LGTM Stack (Loki, Grafana, Tempo, Mimir)*. [https://grafana.com/docs/grafana/latest/](https://grafana.com/docs/grafana/latest/)
- **[3]** *Trivy — Container Vulnerability Scanner Documentation*. [https://trivy.dev/latest/docs/](https://trivy.dev/latest/docs/)
- **[4]** *Google SRE Book — Service Level Objectives (SLI/SLO/Error Budget Methodology and Multi-window Alert Model)*. [https://sre.google/sre-book/service-level-objectives/](https://sre.google/sre-book/service-level-objectives/)
- **[5]** *Jaeger Distributed Tracing Documentation — OTLP/gRPC Ingestion Pipeline, Trace Storage, and UI Query Interface*. [https://www.jaegertracing.io/docs/](https://www.jaegertracing.io/docs/)
