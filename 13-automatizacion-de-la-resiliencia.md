# Parte 3: Escalado, Maduración y Evolución de su Plataforma

## Capítulo 13: Automatización de la Resiliencia

Durante la optimización del gasto en la nube de NewTech, María descubrió que sus equipos de ingeniería aprovisionaban recursos de forma conservadora por temor a superar los controles de costes. Aunque las técnicas de optimización financiera funcionaron, faltaba un componente esencial: **no es posible ahorrar dinero en sistemas que no son fiables**. En el momento en que ocurre una interrupción en producción, los ahorros acumulados se desvanecen ante la fuga de clientes (*churn*) y los costes operativos derivados de la respuesta a incidentes.

La automatización de la resiliencia consiste en contrastar los compromisos de fiabilidad frente a fallos reales inducidos de forma controlada. Es la diferencia entre desear que los sistemas permanezcan en línea y garantizar técnicamente que sobrevivirán a fallos catastróficos. Este enfoque trasciende la monitorización reactiva tradicional para adoptar una **resiliencia proactiva**, definiendo umbrales de fallo aceptables e induciendo interrupciones planificadas mediante ingeniería del caos.

Al finalizar este capítulo, serás capaz de:
- Definir Objetivos de Nivel de Servicio (**SLO**) y publicarlos mediante paneles de observabilidad.
- Implementar procedimientos de copia de seguridad y restauración automatizados con **Velero**.
- Diseñar y ejecutar experimentos de **ingeniería del caos** con **Chaos Mesh** para validar la tolerancia a fallos.
- Configurar estrategias de recuperación ante desastres (**DR**) midiendo **RTO** (*Recovery Time Objective*) y **RPO** (*Recovery Point Objective*).
- Integrar métricas de resiliencia en los procesos de mejora continua de la plataforma.

---

### Definición y Publicación de SLOs

Desde una perspectiva de negocio sin restricciones presupuestarias, el objetivo de disponibilidad teórico sería el 100%. Sin embargo, dado que el coste crece exponencialmente al buscar niveles extremos de disponibilidad, los SLOs permiten asumir márgenes de fallo tolerables que reducen radicalmente los costes operativos.

Un **SLO** representa el objetivo cuantitativo de fiabilidad de un servicio durante una ventana de tiempo (mensual o anual). La diferencia entre tres nueves (99.9% / ~43 minutos de inactividad al mes), cuatro nueves (99.99% / ~4 minutos al mes) y cinco nueves (99.999% / ~26 segundos al mes) se traduce en un incremento masivo en esfuerzo de ingeniería e infraestructura [1].

| Concepto | Definición | En la Práctica | Ejemplos |
| :--- | :--- | :--- | :--- |
| **Indicador de Nivel de Servicio (*SLI*)** | Medición del comportamiento real de tu servicio | Tiempo de respuesta de API, tasa de error, tiempo de actividad, proporción de solicitudes exitosas | La latencia en el percentil 95 de la API es de 420 ms.<br>El 99.7% de las solicitudes tuvieron éxito en los últimos 30 días. |
| **Objetivo de Nivel de Servicio (*SLO*)** | Un objetivo cuantitativo para el SLI | Objetivo de latencia, objetivo de disponibilidad, presupuesto de error (*error budget*) | El 99.9% de las solicitudes se completan en menos de 500 ms en una ventana móvil de 30 días.<br>El tiempo de actividad mensual debe ser de al menos 99.95%. |
| **Acuerdo de Nivel de Servicio (*SLA*)** | Un compromiso formal con los clientes, generalmente con penalizaciones por incumplimiento | Garantía contractual de disponibilidad con créditos de servicio | Garantizamos un 99.9% de disponibilidad mensual. Si caemos por debajo de esto, los clientes reciben un crédito de servicio del 10%. |

> **Tabla 13.1** - Conceptos fundamentales de niveles de servicio

#### Presupuesto de Error (*Error Budget*)
Representa el margen de fallo permitido antes de vulnerar el SLO. Si el objetivo es 99.9%, el presupuesto de error es del 0.1% (~43 minutos/mes). Permite:
- **Congelación de Despliegues (*Deployment Freeze*)**: Si el presupuesto se agota, se pausan nuevos lanzamientos para priorizar la estabilidad.
- **Equilibrio entre Riesgo y Velocidad**: Permite a los equipos acelerar cambios mientras conserven presupuesto.
- **Alineación Organizacional**: Establece un criterio cuantitativo común entre negocio e ingeniería.

> Estándares abiertos como **OpenSLO** [6] permiten definir SLOs de forma declarativa e independiente del proveedor para Prometheus, Datadog o New Relic.

---

### Implementación Práctica de SLOs con Sloth

**Sloth** [2] genera automáticamente reglas de grabación (*recording rules*) y alertas para Prometheus a partir de manifiestos declarativos:

```yaml
version: "1"
service: "auth-service"
labels:
  team: platform-engineering
objectives:
  - name: availability
    description: "Percentage of successful authentication requests"
    indicator:
      events: # Availability SLI: ratio of successful requests
        errorQuery: sum(rate(http_requests_total{job="auth-service",code=~"(5..)"}[{{.window}}]))
        totalQuery: sum(rate(http_requests_total{job="auth-service"}[{{.window}}]))
    slo: 99.95
    alerting:
      name: auth_slo_alert
      labels:
        severity: critical
  - name: latency
    description: "95th percentile latency under 200 ms"
    indicator:
      latency:
        - name: "auth_request_duration_seconds"
          buckets:
            - 0.05
            - 0.1
            - 0.2
    slo: 95
    alerting:
      name: auth_latency_slo_alert
```

Generación y despliegue de reglas de Prometheus:

```bash
# Sloth genera las reglas de grabación y alertas
sloth generate -i slos.yaml -o rules.yaml
```

Métricas calculadas automáticamente:
- `slo:auth_service_availability:ratio_rate_5m`
- `slo:auth_service_availability:error_budget_rate`
- `slo:auth_service_availability:error_budget_remaining`

---

### Automatización de Copias de Seguridad y Restauración con Velero

**Velero** [3] respalda el estado completo del clúster de Kubernetes: recursos declarativos (CRDs, Secrets, ConfigMaps) e instantáneas (*snapshots*) de volúmenes persistentes vía CSI hacia almacenamiento de objetos (S3/GCS).

> **Figura 13.1** - Ruta automatizada de copia de seguridad y restauración con Velero

#### 1. Script de Validación de Restauración en Clúster de Pruebas

```bash
#!/bin/bash
LATEST_BACKUP=$(velero backup get --sort-by=.metadata.creationTimestamp | tail -1)

# Check if backup was found
if [ -z "$LATEST_BACKUP" ]; then
    echo "Error: No backups found"
    exit 1
fi

# Restore to test environment
velero restore create --from-backup $LATEST_BACKUP \
  --namespace-mappings production=test-restore \
  --include-namespaces production

# Run validation checks
kubectl get pods -n test-restore
kubectl logs -n test-restore -l app=api --tail=50

# Verify data integrity
kubectl exec -n test-restore deployment/validator -- \
  /scripts/validate-data.sh
```

#### 2. Configuración de Respaldos en Autoservicio mediante Etiquetas

Configuración del chart de Helm (`values.yaml`):

```yaml
# values.yaml in platform-backup-config chart
backup:
  enabled: true
  schedule: "0 3 * * *" # Daily at 3 AM
  retention: "30d"
  selectors:
    - matchLabels:
        backup: enabled
  excludedNamespaces:
    - kube-system
    - velero
```

Los desarrolladores activan respaldos etiquetando sus recursos:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    backup: enabled
data:
  config: |
    key: value
```

---

### Ingeniería del Caos con Chaos Mesh

La ingeniería del caos prueba la resiliencia del sistema introduciendo fallos deliberados y controlados para descubrir debilidades antes de que ocurran incidentes reales.

> **Figura 13.2** - Ciclo de vida de un experimento con Chaos Mesh e integración con monitorización

#### Experimento de Latencia de Red Programada (`NetworkChaos`)

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: api-latency-test
  namespace: production
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: api-service
  delay:
    latency: "100ms"
    jitter: "10ms"
  duration: "5m"
  scheduler:
    cron: "0 9 * * 1" # Every Monday at 9 AM
```

#### Criterios para la Planificación de Experimentos de Caos:
- **Horario laboral (9 AM – 5 PM)**: Garantiza la disponibilidad de los ingenieros de guardia para supervisar y actuar.
- **Programación predecible mediante cron**: Evita sorpresas comunicando los calendarios a los equipos.
- **Fuera de horas punta**: Mitiga el impacto en los usuarios finales durante pruebas iniciales.

---

### Recuperación ante Desastres Multi-Región (*Disaster Recovery*)

> **Figura 13.3** - Arquitectura de recuperación ante desastres multi-región y ruta de conmutación por error (*failover*)

La arquitectura multi-región implementa una región primaria activa (`us-east-1`) con conmutación mediante Route 53 hacia una región secundaria en modo *hot standby* (`us-west-2`) respaldada por réplicas de lectura en RDS y replicación de almacenamiento S3 con Velero, garantizando tiempos de conmutación (*RTO*) inferiores a 15 minutos.

#### Pruebas de Caos en Servicios Gestionados frente a Kubernetes:
- **Servicios Gestionados (RDS, S3, DynamoDB)**: Pruebas de solo lectura (simulación de consultas lentas, saturación de conexiones, conmutación Multi-AZ forzada y latencia entre regiones).
- **Cargas de Trabajo en Kubernetes**: Inyección activa de fallos (eliminación de pods con `PodChaos`, fallos de red con `NetworkChaos` y estrés de CPU/memoria con `ResourceChaos`).

---

### Ejercicio 13.1: Despliegue de Chaos Mesh y Prueba de Eliminación de Pods

1. **Instalar Chaos Mesh**: Desplegar el controlador y el panel mediante Helm en el namespace `chaos-mesh`.
2. **Desplegar la aplicación objetivo**: Crear un despliegue de 3 réplicas de Nginx en el namespace `chaos-testing`.
3. **Aplicar el experimento de fallo de pods**: Ejecutar `chaos-mesh-pod-failure.yaml` para probar la eliminación de pods individuales y el 50% de réplicas simultáneas.
4. **Supervisar la autorreparación (*self-healing*)**: Observar en tiempo real cómo el controlador ReplicaSet reemplaza los pods terminados en cuestión de segundos.
5. **Monitorizar en Prometheus y Grafana**: Analizar las métricas de la familia `chaos_mesh_*` junto a la tasa de reinicios del clúster.

---

### Resumen

- **Resiliencia Proactiva**: Probar continuamente los sistemas bajo condiciones reales de fallo en lugar de confiar pasivamente en la redundancia teórica.
- **Gestión Cuantitativa de SLOs**: Herramientas como Sloth convierten objetivos de disponibilidad y latencia en reglas de grabación y alertas en Prometheus.
- **Presupuestos de Error Operativos**: Guían las decisiones de lanzamiento y congelación de despliegues equilibrando velocidad y estabilidad.
- **Copias de Seguridad Automatizadas con Velero**: Respaldan recursos declarativos y volúmenes persistentes, requiriendo pruebas de restauración periódicas.
- **Ingeniería del Caos con Chaos Mesh**: Descubre dependencias ocultas y fallos de conmutación mediante la inyección controlada de latencia y caída de pods.

---

### Referencias

- **[1]** *Uptime and Downtime Availability Calculator (Three Nines)*. [https://hyperping.com/three-nines](https://hyperping.com/three-nines)
- **[2]** *Sloth — Modern and Easy SLO Generator for Prometheus*. [https://sloth.dev/](https://sloth.dev/)
- **[3]** *Velero — Backup and Migrate Kubernetes Resources and Persistent Volumes*. [https://velero.io/](https://velero.io/)
- **[4]** *Netflix Chaos Monkey*. [https://netflix.github.io/chaosmonkey/](https://netflix.github.io/chaosmonkey/)
- **[5]** *Chaos Mesh — A Powerful Chaos Engineering Platform for Kubernetes*. [https://chaos-mesh.org/](https://chaos-mesh.org/)
- **[6]** *OpenSLO Specification*. [https://openslo.com](https://openslo.com/)
