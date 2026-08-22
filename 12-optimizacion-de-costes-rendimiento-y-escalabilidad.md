# Parte 3: Escalado, Maduración y Evolución de su Plataforma

## Capítulo 12: Optimización de Costes, Rendimiento y Escalabilidad

Una vez construidos los componentes esenciales de la plataforma de ingeniería (seguridad, gobernanza, fiabilidad y resiliencia), tanto los ingenieros de plataforma como los desarrolladores de producto se enfrentan a una pregunta crucial: ¿cómo gestionamos y entendemos el consumo real de recursos? En la era de los recursos ilimitados en la nube, el control económico rara vez se prioriza desde el inicio. Así como aprendimos a aplicar políticas sobre la infraestructura, en este capítulo aprenderemos a hacer que esas políticas sean **económicamente sostenibles**.

En NewTech, el equipo de María vivió un punto de inflexión: los clústeres de Kubernetes operaban con estabilidad y seguridad, pero la factura mensual de AWS generó una alerta inmediata en la dirección financiera (CFO). La orquestación de contenedores abstrae la infraestructura, pero también oculta decisiones costosas: pods con reservas excesivas de memoria, volúmenes persistentes huérfanos y aplicaciones escalando sin límites definidos.

La optimización de costes consiste en entender el destino del gasto, adoptar compromisos intencionados y automatizar procesos para que los desarrolladores entreguen valor sin perseguir anomalías en la facturación. Esta es la esencia de **FinOps** [1] (la convergencia entre finanzas y operaciones).

Al finalizar este capítulo, serás capaz de:
- Comprender las compensaciones arquitectónicas (*trade-offs*) entre coste, rendimiento y escalabilidad.
- Instrumentar la plataforma con herramientas de observabilidad de costes (**OpenCost / Kubecost**).
- Implementar estrategias de escalado automático horizontal (**HPA**) y vertical (**VPA**).
- Ajustar el dimensionamiento de cargas de trabajo (*rightsizing*), utilizar instancias Spot y descuentos por uso comprometido (**CUD**), y seleccionar tipos de instancia eficientes con **Karpenter**.
- Configurar salvaguardas de gobernanza, asignación de costes por equipo y alertas ante anomalías.
- Integrar la optimización financiera en las canalizaciones de CI/CD y en el ciclo de vida de las aplicaciones.

---

### Compensaciones entre Coste, Rendimiento y Escalabilidad

No es posible optimizar simultáneamente coste, rendimiento y escalabilidad al máximo nivel. La labor del ingeniero de plataforma es hacer visibles y justificables estas compensaciones mediante cuatro pilares:

1. **Definir SLOs (*Service Level Objectives*)**: Promesas contractuales cuantificables (ej. 99.9% de disponibilidad, latencia p99 < 100 ms).
2. **Coste por SLO**: Calcular el coste financiero necesario para satisfacer cada SLO, transformando el debate de "esta infraestructura es cara" a "este nivel de servicio cuesta X".
3. **Ajuste de Dimensionamiento (*Rightsizing*)**: Asignar perfiles de cómputo específicos (optimizados para CPU, memoria o E/S) según la naturaleza de la carga.
4. **Automatización**: Eliminar la gestión manual de capacidad mediante controladores dinámicos.

---

### Ejercicio 12.1: Comparativa de Escenarios de Disponibilidad y Coste

Evaluación de tres arquitecturas para una aplicación web sin estado:

| Parámetro | Escenario 1: Optimizado para Coste | Escenario 2: Alta Disponibilidad (HA) | Escenario 3: HA Balanceada + Spot |
| :--- | :--- | :--- | :--- |
| **Configuración** | 1 AZ, 1 × `m5.large` (2 vCPU, 8 GB) | 3 AZ, 3 × `m5.xlarge` (4 vCPU, 16 GB) | 3 AZ, 1 × `m5.xlarge` (Bajo demanda) + 2 × `m5.xlarge` (Spot) |
| **Coste Mensual** | ~$70 / mes ($0.096/h) | ~$420 / mes ($0.576/h) | ~$222 / mes ($0.304/h) |
| **SLO de Disponibilidad** | 99.0% (56 min inactividad/mes) | 99.9% (8.7 min inactividad/mes) | 99.9% (con gestión de interrupciones Spot) |
| **Concurrencia Máxima** | ~500 peticiones (sin HPA) | ~3,000 peticiones (HPA en 3 nodos) | ~3,000 peticiones (HPA en 3 nodos) |
| **Latencia p99** | Degrada sobre 300 RPS | Estable hasta 1,000 RPS | Estable hasta 1,000 RPS |
| **Coste por 1M Peticiones**| ~$0.14 | ~$0.42 | ~$0.22 |

> **Conclusión FinOps**: El Escenario 3 ofrece un **47% de ahorro** frente al Escenario 2 ($222 frente a $420) manteniendo idéntica disponibilidad y rendimiento. La eficiencia de costes se mide en función del valor y el SLO entregado: una instancia barata de $70/mes que cause $50,000 en pérdidas por caídas del servicio resulta mucho más costosa que una alternativa de $420/mes.

---

### Observabilidad Financiera (*FinOps Observability*)

La Fundación FinOps establece tres fases iterativas: **Informar (*Inform*)**, **Optimizar (*Optimize*)** y **Operar (*Operate*)**, complementadas con el marco de las 4R (*Report, Recommend, Remediate, Retain*).

> **Figura 12.1** - Modelo iterativo de FinOps

#### Asignación de Costes con OpenCost mediante Etiquetas de Kubernetes

**OpenCost** [4] (y su versión comercial **Kubecost** [5]) realiza la atribución de costes analizando etiquetas (*labels*) y anotaciones:

1. Asignación a nivel de *Namespace*:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-checkout
  labels:
    team: checkout
    cost-center: "4521"
    business-unit: payments
```

2. Propagación en el *Deployment*:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
  namespace: team-checkout
spec:
  template:
    metadata:
      labels:
        app: checkout-api
        team: checkout
        cost-center: "4521"
```

OpenCost expone métricas en Prometheus como `container_cpu_hourly_cost` y `container_memory_hourly_cost`, permitiendo crear cuadros de mando en Grafana por equipo, centro de costes o unidad de negocio.

---

### Estrategias de Escalado Automático (*Autoscaling*)

> **Figura 12.2** - Comparativa conceptual: Escalado Horizontal (HPA) frente a Escalado Vertical (VPA)

#### 1. Horizontal Pod Autoscaling (HPA)

Ajusta dinámicamente el número de réplicas en función del consumo de recursos o métricas personalizadas de negocio:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-api-hpa
  namespace: team-checkout
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

- `stabilizationWindowSeconds`: Evita fluctuaciones rápidas (*thrashing*) estabilizando las reducciones de escala durante 300 segundos.
- Escalado predictivo por métricas de Prometheus (mediante Prometheus Adapter):

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "500"
```

#### 2. Vertical Pod Autoscaling (VPA)

Ajusta los límites y solicitudes de CPU y memoria de cada contenedor individual basándose en el historial real de uso:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: checkout-api-vpa
  namespace: team-checkout
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-api
  updatePolicy:
    updateMode: "Auto"
    minReplicas: 2
  resourcePolicy:
    containerPolicies:
      - containerName: checkout-api
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: 2
          memory: 2Gi
        controlledResources: ["cpu", "memory"]
      - containerName: sidecar
        controlledResources: []
```

> **Buenas Prácticas**: Utilizar herramientas como **Goldilocks** [6] para evaluar recomendaciones de VPA en modo pasivo antes de activar el modo automático.

---

### Dimensionamiento, Instancias Spot y Gestión de Nodos con Karpenter

#### Provisión Inteligente con Karpenter (`NodePool`)

**Karpenter** [7] evalúa directamente las solicitudes de los pods pendientes y aprovisiona el tipo de instancia EC2 más económica disponible en tiempo real:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["m", "c"]
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m5", "m6i", "c5"]
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["large", "xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      disruption:
        expireAfter: 720h
```

#### Configuración de Tolerancias para Cargas de Trabajo en Instancias Spot

1. Nodo con contaminación (*taint*) de Spot:

```yaml
apiVersion: v1
kind: Node
metadata:
  name: node-spot-1
spec:
  taints:
    - key: instance-type
      value: spot
      effect: NoSchedule
```

2. Despliegue con tolerancia y afinidad hacia capacidad Spot:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  replicas: 1
  selector:
    matchLabels:
      app: batch-processor
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      tolerations:
        - key: instance-type
          operator: Equal
          value: spot
          effect: NoSchedule
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: karpenter.sh/capacity-type
                    operator: In
                    values:
                      - spot
      containers:
        - name: batch-processor
          image: your-image:latest
```

---

### Gobernanza de Costes y Control de Cuotas

#### 1. Cuotas de Recursos y Rangos Límite (`ResourceQuota` y `LimitRange`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: team-analytics
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: analytics-quota
  namespace: team-analytics
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "50"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-analytics
spec:
  limits:
    - type: Container
      min:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "2"
        memory: "4Gi"
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
```

#### 2. Políticas de Validación de Solicitudes con Kyverno [8]

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-requests
spec:
  validationFailureAction: audit
  rules:
    - name: validate-requests
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "CPU and memory requests are required."
        pattern:
          spec:
            containers:
              - resources:
                  requests:
                    memory: "?*"
                    cpu: "?*"
```

---

### Optimización de Costes en la Canalización de CI/CD

Integrar la estimación de costes directamente en los *Pull Requests* genera una cultura financiera proactiva antes de desplegar en producción:

```yaml
# En tu canalización CI/CD (GitHub Actions / GitLab CI):
- name: Check resource cost delta
  run: |
    python3 scripts/cost-check.py \
      --current manifests/deployment.yaml \
      --threshold 200 # Alerta si el incremento supera el 200%
```

Si el despliegue excede el presupuesto del namespace asignado, la canalización se detiene con un mensaje descriptivo:

```text
ERROR: Deployment checkout-api would exceed the namespace budget.
Requested: 2400m CPU (namespace limit: 2000m)
Action: Request a quota increase via the platform-team channel or reduce replicas.
```

---

### Ejercicio 12.3: Reducción del 30% en el Coste de una Aplicación

1. **Establecer la línea base**: Determinar el coste horario inicial de la aplicación con OpenCost.
2. **Perfilar el consumo de recursos**: Monitorizar CPU y memoria reales durante 1 semana con `kubectl top` o Prometheus.
3. **Ajustar el dimensionamiento**: Reducir solicitudes sobredimensionadas manteniendo margen de seguridad (*headroom*).
4. **Implementar HPA**: Configurar escalado horizontal al 75% de utilización y someter a pruebas de carga (`hey`/`wrk`).
5. **Evaluar VPA y Goldilocks**: Analizar recomendaciones automáticas de memoria y CPU.
6. **Optimizar el tipo de instancia**: Seleccionar familias optimizadas para cómputo o memoria según el cuello de botella.
7. **Medir y documentar**: Comparar el gasto tras 2 semanas y publicar los resultados en el portal de la plataforma.

---

### Resumen

- **Compensaciones Fundamentales**: No es posible maximizar simultáneamente coste, rendimiento y escalabilidad; los SLOs cuantifican el coste necesario para cada nivel de servicio.
- **Visibilidad Integral con OpenCost**: Etiquetar namespaces y pods permite realizar asignación de costes por equipo (*showback/chargeback*).
- **Autoscaling como Palanca Principal**: Combinar HPA para fluctuaciones de tráfico y VPA para ajustar el tamaño de réplicas individuales.
- **Aprovisionamiento Dinámico con Karpenter**: Selecciona tipos de instancia y mezcla capacidad Spot con instancias bajo demanda de forma automática.
- **Gobernanza y FinOps Shift-Left**: Validar cuotas en CI/CD y aplicar políticas preventivas antes de que los costes lleguen a la factura final.

---

### Referencias

- **[1]** *FinOps Foundation Official Website*. [https://finops.org/](https://finops.org/)
- **[2]** *Top FinOps Tools for Platform Engineers (2026)*. [https://platformengineering.org/blog/10-finops-tools-platform-engineers-should-evaluate-for-2026](https://platformengineering.org/blog/10-finops-tools-platform-engineers-should-evaluate-for-2026)
- **[3]** *Maximizing Value with Cloud FinOps (Thoughtworks)*. [https://www.thoughtworks.com/en-us/insights/articles/maximizing-value-cloud-finops](https://www.thoughtworks.com/en-us/insights/articles/maximizing-value-cloud-finops)
- **[4]** *OpenCost Documentation*. [https://opencost.io/](https://opencost.io/)
- **[5]** *Kubecost — Kubernetes Cost Monitoring*. [https://www.apptio.com/products/kubecost/](https://www.apptio.com/products/kubecost/)
- **[6]** *Goldilocks — Right-sizing Kubernetes Deployments*. [https://www.fairwinds.com/blog/introducing-goldilocks-a-tool-for-recommending-resource-requests](https://www.fairwinds.com/blog/introducing-goldilocks-a-tool-for-recommending-resource-requests)
- **[7]** *Karpenter — Kubernetes Node Autoscaling*. [https://karpenter.sh/](https://karpenter.sh/)
- **[8]** *Kyverno — Kubernetes Native Policy Management*. [https://kyverno.io/](https://kyverno.io/)
