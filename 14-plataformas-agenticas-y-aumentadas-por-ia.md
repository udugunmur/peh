# Parte 3: Escalado, Maduración y Evolución de su Plataforma

## Capítulo 14: Plataformas Agénticas y Aumentadas por IA

A lo largo de los capítulos anteriores hemos abordado desde los cimientos y componentes esenciales hasta el escalado y resiliencia de las plataformas de ingeniería. Como etapa culminante, exploraremos cómo las **plataformas agénticas y aumentadas por Inteligencia Artificial** están redefiniendo la disciplina: reduciendo la carga cognitiva de los desarrolladores y potenciando la capacidad de los ingenieros de plataforma para operar a gran escala.

La transición desde operaciones guiadas por observabilidad hacia plataformas aumentadas por IA representa una maduración cualitativa. En NewTech, el equipo de María enfrentó un desafío representativo: a pesar de recopilar ingentes volúmenes de métricas, trazas y registros, seguían escapando incidentes críticos debido a la incapacidad humana para correlacionar señales dispersas en tiempo real y ejecutar remediaciones inmediatas. La IA actúa aquí como una **capa de razonamiento sobre los datos operativos**, permitiendo correlacionar anomalías, contextualizar fallos con el historial técnico y ejecutar acciones correctivas bajo supervisión humana controlada.

Al finalizar este capítulo, serás capaz de:
- Integrar herramientas de IA generativa en flujos de ingeniería sin depender exclusivamente de APIs propietarias en la nube.
- Diseñar sistemas multi-agente con clara separación de responsabilidades y niveles de supervisión.
- Implementar **RAG (*Retrieval-Augmented Generation*)** para dotar a la plataforma de documentación técnica inteligente y contextualizada.
- Construir **salvaguardas (*guardrails*)** y mecanismos de intervención humana en el bucle (*human-in-the-loop*).
- Evaluar el retorno de inversión (**ROI**), la reducción del **MTTR** y el impacto en la productividad del desarrollador.

---

### Casos de Uso y Criterios de Adopción de IA en la Plataforma

Las organizaciones que tienen éxito con plataformas aumentadas por IA comienzan identificando cuellos de botella operativos concretos, en lugar de preguntar qué construir con los modelos.

> **Figura 14.1** - Puntos de integración de IA a lo largo del stack de ingeniería de plataformas

#### Cuándo aplicar IA en Plataformas:
- **Tareas cognitivas repetitivas de alto volumen**: Deduplicación y triaje de alertas, búsqueda y síntesis de documentación técnica.
- **Reconocimiento de patrones complejos**: Detección de anomalías multidimensionales y correlación de eventos en tiempo real.
- **Priorización y asignación contextual**: Enrutamiento automático de incidentes según el historial y la topología del servicio.
- **Interfaces en lenguaje natural**: Consultas operativas y generación declarativa de recursos sin fricción sintáctica.

#### Dónde evitar la autonomía de la IA (por ahora):
- Decisiones arquitectónicas fundacionales.
- Validaciones regulatorias y legales estrictas sin auditoría determinista.
- Acciones destructivas o irreversibles (borrado de datos, cambios masivos de IAM) sin aprobación humana explícita.

---

### Enfoques de Despliegue de Modelos: APIs Gestionadas frente a Open Source

| Dimensión | APIs Gestionadas (GPT-4, Gemini, Claude) | OSS Alojado en la Nube (Llama, Mistral vía Bedrock/Vertex) | OSS Autoalojado (Llama, Mistral vía Ollama) |
| :--- | :--- | :--- | :--- |
| **Calidad de razonamiento** | Alta | Buena para tareas acotadas y específicas de dominio | Buena para tareas acotadas y específicas de dominio |
| **Infraestructura** | Ninguna requerida | Ninguna requerida | Se requiere GPU o CPU cuantizada |
| **Privacidad de datos** | Los datos salen de tu infraestructura | Permanece dentro de tu cuenta cloud | Los datos se mantienen locales |
| **Modelo de costes** | Por token, continuo | Por token | Costes de GPU continuos (servicio + reentrenamiento) |
| **Control** | Limitado | Elección de modelo; requiere cierta configuración | Control total del stack |
| **Ideal para** | Razonamiento complejo, prototipado, datos públicos | Entornos regulados que requieren modelos OSS sin ops de GPU | Control máximo, entornos air-gapped, sensibilidad a costes con alto volumen |

> **Tabla 14.1** - Comparativa de enfoques de despliegue de modelos de IA para ingeniería de plataformas

> **Estrategia Híbrida Recomendada**: Utilizar modelos de código abierto autoalojados (mediante **Ollama** [8], **vLLM** o **TensorRT-LLM**) para tareas de alta frecuencia y baja latencia (triaje inicial, embeddings de búsqueda, análisis de logs) y APIs gestionadas avanzadas para decisiones complejas que requieran razonamiento profundo.

---

### Patrones de Integración en Flujos de Trabajo de Desarrollo

#### 1. Generación Asistida de Canalizaciones de CI/CD mediante RAG
El desarrollador describe en lenguaje natural los requisitos de su microservicio y la infraestructura requerida. Un asistente RAG recupera plantillas probadas, políticas de seguridad y configuraciones canarias, generando manifiestos conformes sin alucinaciones.

Requisitos indispensables para la ejecución de canalizaciones generadas por IA:
- **Validación en simulacro (*Dry-Run*)**: Probar los pasos en un entorno aislado (*sandbox*).
- **Inspección de políticas**: Validar con OPA Gatekeeper o Kyverno antes de la admisión.
- **Puerta de aprobación humana (*Human Approval Gate*)**: Revisión explícita de comandos y radio de impacto (*blast radius*).

#### 2. Bots de Triaje y Correlación de Incidentes
El agente recopila logs recientes, métricas de latencia y errores, consulta la base de incidentes pasados y sugiere hipótesis de causa raíz con puntuaciones de confianza.

```python
def correlate(self, alerts: Optional[List[Alert]] = None) -> List[CorrelatedIncident]:
    """
    Correlate pending alerts into incidents.
    Args:
        alerts: Optional list of alerts (uses pending_alerts if None).
    Returns:
        List of correlated incidents
    """
    if alerts:
        self.pending_alerts = alerts
    if not self.pending_alerts:
        return []
    
    # Group by time windows
    time_groups = self._group_by_time()
    incidents = []
    
    for group in time_groups:
        # Further correlate within time group
        correlations = self._correlate_group(group)
        for correlation in correlations:
            incident = self._create_incident(correlation)
            self._analyze_root_cause(incident)
            incidents.append(incident)
            
    self.incidents.extend(incidents)
    self.pending_alerts = []
    return incidents
```

#### 3. RAG para Documentación Inteligente de la Plataforma
Convierte la documentación estática en un sistema interactivo indexando guías de arquitectura (ADRs), runbooks y post-mortems en una base de datos vectorial como **ChromaDB** [11].

> **Recuperación Híbrida**: Combinar búsqueda vectorial semántica (60% de peso) con búsqueda por palabras clave BM25 (40% de peso) garantiza recuperar tanto conceptos abstractos como comandos exactos (ej. `kubectl rollout undo`).

Despliegue declarativo de ChromaDB en Kubernetes con Pulumi:

```python
import pulumi_kubernetes as k8s

chromadb = k8s.apps.v1.Deployment(
    "chromadb",
    metadata={"name": "chromadb", "namespace": "ai-platform"},
    spec={
        "replicas": 1,
        "selector": {"matchLabels": {"app": "chromadb"}},
        "template": {
            "metadata": {"labels": {"app": "chromadb"}},
            "spec": {
                "containers": [{
                    "name": "chromadb",
                    "image": "chromadb/chroma:latest",
                    "ports": [{"containerPort": 8000}],
                    "volumeMounts": [{"name": "data", "mountPath": "/chroma/chroma"}],
                }],
                "volumes": [{
                    "name": "data",
                    "persistentVolumeClaim": {"claimName": "chromadb-pvc"},
                }],
            },
        },
    },
)
```

---

### Diseño de Sistemas Multi-Agente

Un **agente** es un sistema autónomo que recibe instrucciones, razona, ejecuta acciones mediante herramientas, observa resultados y ajusta su comportamiento.

> **Figura 14.2** - Arquitectura de plataforma multi-agente: roles, límites de supervisión y flujos de interacción

#### Roles Fundamentales en el Ecosistema de Plataforma:
1. **Agente de Triaje (*Triage Agent*)**: Primer respondedor ante alertas; clasifica incidentes, asigna equipos y abre tickets.
2. **Agente de Remediación (*Remediation Agent*)**: Ejecuta pasos de runbooks aprobados (reinicios, reversiones, escalado).
3. **Agente de Documentación (*Documentation Agent*)**: Detecta brechas de conocimiento y actualiza runbooks tras cada incidente.
4. **Agente de Observabilidad (*Observability Agent*)**: Calibra umbrales de alerta y detecta anomalías en segundo plano.
5. **Agente de Mejora de Plataforma (*Platform Improvement Agent*)**: Analiza tendencias de MTTR y costes para proponer optimizaciones en los *golden paths*.

| Nivel de Riesgo | Quién Decide | Quién Ejecuta | Ejemplos |
| :--- | :--- | :--- | :--- |
| **Seguro (Autónomo)** | Agente | Agente | Creación de tickets de incidentes, actualización de documentación, consultas de solo lectura (logs, métricas), ejecución de pruebas de humo, recolección de contexto y correlación de eventos |
| **Medio (Aprobación humana)** | El agente propone, el humano aprueba | Agente | Reinicio de servicios, reversión (*rollback*) de despliegues recientes, escalado de infraestructura, actualización de umbrales de alerta, cambios de configuración |
| **Alto (Decisión humana)** | Humano | Humano | Destrucción de datos o copias de seguridad, modificación de políticas de seguridad o infraestructura crítica, migraciones de bases de datos, compromisos financieros |

> **Tabla 14.2** - Clasificación de acciones del agente y tolerancia al riesgo

| Métrica | Qué Mide | Estado Saludable | Requiere Investigación |
| :--- | :--- | :--- | :--- |
| **Puntuación de Confianza (*Confidence score*)** | Nivel de certeza del agente en su decisión | 75–95% | Consistentemente por debajo del 60% |
| **Tasa de Anulación Humana (*Human override rate*)** | Frecuencia con la que los ingenieros rechazan las recomendaciones | Inferior al 5% | Superior al 30% (el agente propone decisiones deficientes) |
| **Precisión de Triaje (*Accuracy*)** | Porcentaje de incidentes clasificados correctamente | Superior al 85% | Inferior al 80% (detener el uso autónomo) |
| **Tiempo de Resolución (*MTTR*)** | Tiempo medio de resolución asistido por IA frente a manual | Más rápido con IA | Más lento con IA (revisar la calidad de la señal) |
| **Tasa de Falsos Positivos** | Frecuencia con la que el agente alerta sobre falsos problemas | Inferior al 10% | Superior al 25% (recalibrar umbrales) |
| **Coste por Acción** | Infraestructura, tokens y tiempo humano por resolución | Estable o decreciente | Disparos repentinos (revisar consumo de tokens o bucles infinitos) |

> **Tabla 14.3** - Criterios de evaluación de salud de los agentes

> **Estándares y Frameworks**: En producción, utiliza frameworks consolidados como **LangGraph** (LangChain), **CrewAI**, **AutoGen** (Microsoft) o el **Anthropic Agent SDK**. Para la integración desacoplada de herramientas (Kubernetes, GitHub, Prometheus), adopta el estándar **Model Context Protocol (MCP)**.

---

### Salvaguardas (*Guardrails*) y Mitigación de Riesgos

> **Figura 14.3** - Arquitectura del asistente de plataforma impulsado por RAG: capas de recuperación, generación y salvaguardas

| Tipo de Salvaguarda | Regla | Condición de Disparo |
| :--- | :--- | :--- |
| **Límite de Tasa (*Rate limiting*)** | Máximo de X avisos (*pages*) por hora por incidente | El agente notifica repetidamente a los ingenieros por el mismo incidente |
| **Ámbito (*Scope*)** | No producción: reinicio autónomo.<br>Producción: requiere aprobación | El agente de remediación reinicia un servicio |
| **Dependencia** | Verificar la salud de los sistemas dependientes antes de actuar | Los sistemas dependientes están en estado no saludable |
| **Coste** | Autoaprobar por debajo de $YY/hora. Por encima: requerir aprobación | El agente escala la infraestructura |
| **Reversión (*Rollback*)** | Cancelar acciones posteriores y notificar al humano | Una acción del agente se revierte dentro de los primeros N minutos |

> **Tabla 14.4** - Tipos de salvaguardas, reglas y condiciones de disparo

| Riesgo | Descripción | Mitigación |
| :--- | :--- | :--- |
| **Sobre-automatización (*Over-automation*)** | El agente ejecuta de forma autónoma una acción que requería aprobación | Umbrales de confianza conservadores, humano en el bucle para acciones de alto impacto, registro exhaustivo |
| **Degradación del modelo (*Model degradation*)** | El rendimiento se desvía a medida que la distribución de datos cambia con el tiempo | Monitorizar la precisión continuamente, reentrenar periódicamente, alertar cuando la desviación supere el umbral |
| **Inyección de instrucciones (*Prompt injection*)** | Entradas adversarias manipulan el comportamiento del LLM mediante datos de usuario no saneados | Validación estricta de entradas, uso de entradas estructuradas, nunca pasar texto sin sanear a los prompts |
| **Fuga de contexto (*Context leakage*)** | Información confidencial de una organización se filtra a través de la ventana de contexto del LLM | Aislamiento estricto por inquilino (*tenant*), nunca compartir contexto entre límites, auditorías regulares |
| **Alucinación (*Hallucination*)** | El modelo genera información plausible pero incorrecta | Anclar salidas en datos recuperados (RAG), exigir umbrales de confianza y validar antes de la ejecución |

> **Tabla 14.5** - Riesgos principales en plataformas aumentadas por IA y estrategias de mitigación

---

### Gobernanza y Observabilidad de Modelos de IA

| Categoría | Métrica | Qué Informa |
| :--- | :--- | :--- |
| **LLM** | Uso de tokens | Una tendencia al alza puede indicar bucles de razonamiento infinitos |
| **LLM** | Latencia de inferencia | Indica si la cuantización o la carga están degradando el tiempo de respuesta |
| **LLM** | Puntuaciones de confianza | Una caída en las puntuaciones señala desviación (*drift*) del modelo |
| **LLM** | Precisión frente a verdad básica (*ground truth*) | Muestreo de resultados en decisiones de alto volumen para rastrear precisión |
| **LLM** | Coste por acción | Costes de generación de embeddings, inferencia e infraestructura |
| **Usuario Final** | Tasa de anulación humana | Por encima de X%: recalibrar. Por debajo de Y%: considerar ampliar ámbito (X > Y) |
| **Usuario Final** | Tiempo de resolución | MTTR asistido por IA frente a triaje manual |
| **Usuario Final** | Tasa de falsos positivos | Frecuencia con la que el agente reporta problemas inexistentes |
| **Usuario Final** | Satisfacción del usuario | Si la respuesta asistida por IA resulta más o menos efectiva para los equipos |
| **Infraestructura** | Utilización de GPU/CPU | Si la infraestructura local del modelo se aprovecha eficientemente |
| **Infraestructura** | Latencia de consulta a la BD vectorial | RAG solo es útil si la recuperación es rápida |
| **Infraestructura** | Profundidad de la cola de acciones del agente | Si los agentes producen más trabajo del que pueden ejecutar, escalar o limitar |

> **Tabla 14.6** - Métricas de observabilidad para plataformas aumentadas por IA: LLM, usuario final e infraestructura

> Herramientas especializadas como **LangSmith** [14], **Langfuse** [15] y **Helicone** [16] permiten rastrear trazas de ejecución de LLM, consumo de tokens y análisis de latencia p99 sin alterar la lógica de negocio.

---

### Ejercicio 14.1: Construcción de un Runbook Aumentado por IA

1. **Seleccionar un runbook frecuente**: Ej. depuración de fallos de despliegue o degradación de réplicas en bases de datos.
2. **Identificar fases repetitivas**: Diagnósticos iniciales, recolección de logs y comandos de inspección habituales.
3. **Diseñar la capa de IA**: Definir las acciones de recopilación autónoma y los puntos de aprobación humana antes de reiniciar servicios.
4. **Definir la arquitectura RAG**: Conectar la base de datos vectorial de documentación con el LLM y configurar salvaguardas de ejecución.
5. **Calcular el impacto y ROI**: Medir la reducción de tiempo invertido por incidente y evaluar la precisión de las recomendaciones.

---

### Resumen

- **Enfoque Centrado en el Negocio**: La IA debe resolver cuellos de botella operativos reales, no implementarse como un fin en sí misma.
- **Arquitectura Híbrida Pragmática**: Combinar modelos locales de código abierto (Ollama/vLLM) para alta frecuencia con APIs avanzadas para razonamiento crítico.
- **RAG como Pilar de Veracidad**: Anclar el conocimiento en datos reales (documentación, métricas, código) para eliminar alucinaciones.
- **Sistemas Multi-Agente Supervisados**: Delegar tareas autónomas de bajo riesgo y exigir validación humana en acciones destructivas.
- **Salvaguardas Deterministas en Código**: Las restricciones de seguridad y presupuesto deben implementarse en código estricto, nunca confiarse únicamente a las instrucciones del prompt.
- **Observabilidad Integral de IA**: Medir latencia, consumo de tokens, tasa de anulación humana y MTTR para evaluar el valor real entregado a la organización.

---

### Referencias

- **[1]** *OpenAI Models Reference*. [https://platform.openai.com/docs/models](https://platform.openai.com/docs/models)
- **[2]** *OpenAI*. [https://openai.com/](https://openai.com/)
- **[3]** *Anthropic*. [https://www.anthropic.com/](https://www.anthropic.com/)
- **[4]** *Google DeepMind Gemini*. [https://gemini.google/](https://gemini.google/)
- **[5]** *Microsoft Copilot*. [https://copilot.microsoft.com/](https://copilot.microsoft.com/)
- **[6]** *Meta Llama*. [https://www.llama.com/](https://www.llama.com/)
- **[7]** *Mistral AI*. [https://mistral.ai/](https://mistral.ai/)
- **[8]** *Ollama — Get up and running with large language models locally*. [https://ollama.ai/](https://ollama.ai/)
- **[9]** *Retrieval-Augmented Generation (RAG) Architecture*. [https://aws.amazon.com/what-is/retrieval-augmented-generation/](https://aws.amazon.com/what-is/retrieval-augmented-generation/)
- **[10]** *What is a Vector Database?*. [https://developers.cloudflare.com/vectorize/reference/what-is-a-vector-database/](https://developers.cloudflare.com/vectorize/reference/what-is-a-vector-database/)
- **[11]** *ChromaDB — The AI-native open-source embedding database*. [https://www.trychroma.com/](https://www.trychroma.com/)
- **[12]** *Risks and Failure Modes in Autonomous Agent Systems*. [https://huggingface.co/papers/2602.20021](https://huggingface.co/papers/2602.20021)
- **[13]** *The Value of Internal Developer Platform Investments (IT Revolution)*. [https://itrevolution.com/product/value-internal-developer-platform-investments/](https://itrevolution.com/product/value-internal-developer-platform-investments/)
- **[14]** *LangSmith by LangChain*. [https://smith.langchain.com](https://smith.langchain.com/)
- **[15]** *Langfuse — Open Source LLM Engineering Platform*. [https://langfuse.com](https://langfuse.com/)
- **[16]** *Helicone — LLM Observability and Monitoring*. [https://www.helicone.ai](https://www.helicone.ai/)
