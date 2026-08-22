# Parte 1: Diseño, Construcción y Despliegue de la Plataforma de Ingeniería Principal

## Capítulo 1: Ingeniería de Plataformas: Sentando las Bases

Tras una carrera consolidándose como una desarrolladora destacada en plataformas y operaciones, María acaba de conseguir un nuevo empleo como líder de ingeniería de plataformas en NewTech, una empresa emergente (*startup*) que ha alcanzado el éxito y está expandiendo su base de clientes con rapidez. El CTO de la empresa quiere asegurarse de que sus sistemas puedan escalar para satisfacer la creciente demanda, pero también desea garantizar entregas rápidas sin tener que contratar a demasiados desarrolladores nuevos. 

Es un problema difícil de resolver, ya que, aunque la arquitectura de software utiliza microservicios, la mayoría de los equipos de la empresa han crecido de forma independiente, lo que ha generado una escasa estandarización en las pilas tecnológicas y en la forma en que se despliegan las partes de la aplicación. Para empeorar las cosas, casi a diario se identifican nuevos servicios para ser creados y puestos en marcha como nuevos proyectos, y las exigencias de una infraestructura escalable y segura requieren competencias poco comunes en la empresa. María se dio cuenta rápidamente de que ninguna cantidad de "automatización" solucionaría esto, porque con los equipos moviéndose tan rápido y en tantas direcciones, lograr que todos la adoptaran sería casi imposible. La única solución consistiría en sentar las bases de una plataforma de ingeniería que facilitara la vida de los equipos de desarrollo, operaciones y seguridad, independientemente de la escala de las operaciones, de modo que desearan utilizarla.

Existen muchas interpretaciones sobre lo que hace un equipo de Ingeniería de Plataformas, y se debe reconocer que en escenarios comunes como este no se trata solo de automatizar la infraestructura que necesitan los equipos de desarrollo para construir y desplegar aplicaciones de primer nivel. En su lugar, se trata de crear experiencias excepcionales para todas las partes interesadas (*stakeholders*) involucradas en el ciclo de vida de conceptualizar, construir y dar vida a esas aplicaciones. Esto incluye a los desarrolladores, así como la forma en que los equipos interactúan con redes, seguridad, cumplimiento normativo, el negocio y el propio equipo de plataforma. En conjunto, denominamos a esta experiencia global **DevEx** (*Developer Experience* o Experiencia del Desarrollador). Esto significa que, en el contexto de este libro, hablaremos mucho más sobre todas las partes interesadas que participan en el proceso y cómo deben tenerse en cuenta a medida que se construyen estas plataformas.

Tener en cuenta al usuario final a la hora de diseñar, construir y desplegar una plataforma para que los equipos quieran utilizarla no es diferente de construir un producto de software que una empresa lanzaría al mercado, razón por la cual en este libro se enfatiza el concepto de **plataforma como producto** (*platform-as-a-product*). Sin embargo, existen ciertos matices al crear productos internos que debemos establecer desde el principio del desarrollo:

- Primero, queremos considerar al equipo de desarrollo de la plataforma igual que a cualquier otro equipo de la organización. Eso significa que debemos incorporar prácticas que aceleren a los desarrolladores de la plataforma a la vez que facilitan su trabajo.
- También debemos asegurarnos de que la gobernanza y el cumplimiento normativo de la organización estén integrados en el propio desarrollo de la plataforma; de lo contrario, corremos el riesgo de optimizar el resto de la organización pero dejar la plataforma en un estado difícil de mantener.

Podemos alcanzar estos objetivos aplicando prácticas comprobadas de ingeniería de software como fallar rápido (*fail fast*) y acortar los bucles de retroalimentación, al tiempo que desplazamos las pruebas, la gobernanza y el cumplimiento hacia la izquierda (*shift-left*) para entregar tanto con rapidez como con seguridad. Establecer esta base es el objetivo de este [Capítulo 1](https://subscription.packtpub.com/book/programming/9781806380138/2).

Al final de este capítulo, prepararás a tu equipo de plataforma para el éxito desde el Día 1 mediante:

- El establecimiento de un conjunto repetible de prácticas de desarrollo y lanzamiento para el equipo de producto de la plataforma.
- La integración de prácticas rigurosas de desarrollo de software en tus bases de código iniciales.
- La creación de una estructura de base de código mantenible para garantizar una plataforma de ingeniería segura, estable y de alto rendimiento.
- La automatización de la incorporación de nuevos desarrolladores de plataforma y bases de código para reducir la sobrecarga a medida que la plataforma crece.

---

### Principios Fundamentales de Diseño

Los principios de diseño que utilizaremos al trabajar en nuestra plataforma de ingeniería se describen en la Figura 1.1 a continuación. Aquí se presentan seis conceptos clave en el proceso de desarrollo de plataformas que analizaremos en el resto del [Capítulo 1](https://subscription.packtpub.com/book/programming/9781806380138/2) y a lo largo del libro. Cada uno de estos conceptos es esencial para construir una plataforma de ingeniería escalable y eficaz. Debajo de cada concepto se muestran ejemplos de componentes que pueden considerarse parte de él.

> **Figura 1.1** - Conceptos clave en el proceso de desarrollo de la plataforma

La Figura 1.1 resume los principios clave, pero lo que a menudo se pasa por alto es la razón detrás de ellos. Comprender el razonamiento facilitará que los equipos adapten estos conceptos a sus entornos de desarrollo de forma fluida. A diferencia de los principios generales de diseño de software que se centran en la resiliencia y escalabilidad del sistema, los principios de ingeniería de plataformas se orientan hacia la **Experiencia del Desarrollador (DevEx)**, la eficiencia operativa y la adaptabilidad (a veces las tres cosas a la vez). Rara vez se trata de cuánto tiempo permanece en ejecución un único servicio, sino más bien de cómo se puede construir, probar y operar el ecosistema general de servicios. Estos principios enfatizan la parte de DevEx, asegurando que los desarrolladores puedan concentrarse en aportar valor en lugar de lidiar con la configuración de infraestructura y procesos.

Es fundamental establecer estos principios de forma temprana, ya que constituyen la base sobre la que crecerá la plataforma. A medida que la plataforma escala (lo que inevitablemente permite el escalado del negocio), el coste de corregir errores arquitectónicos iniciales aumenta significativamente. Por lo tanto, estos principios tienen como objetivo orientar las decisiones en lugar de actuar como una lista de verificación rígida. Las concesiones (*trade-offs*) son inevitables en la ingeniería de plataformas, pero comprender sus implicaciones permite tomar decisiones deliberadas en lugar de reaccionar ante problemas operativos más adelante. Veamos con mayor detenimiento estos principios a través de tres ejes: **mentalidad de producto**, **higiene de equipo y código**, y **seguridad y cumplimiento**.

#### Mentalidad de Producto (*Product Mindset*)

##### Plataforma como Producto (*Platform as Product*)
Plataforma como producto es el concepto de construir su plataforma con todos los atributos de cualquier producto de software comercial. Esto significa que la plataforma de ingeniería que construya no será un proyecto con una definición fija de terminado (*definition of done*), sino un ciclo de vida de producto con todos los atributos correspondientes, centrado en proporcionar algún tipo de valor al cliente. Para una plataforma de ingeniería, el valor entregado al cliente es la **Experiencia del Desarrollador (DevEx)**. Para ofrecer esto a lo largo del ciclo de vida del producto, necesitará incorporar no solo los comentarios de los desarrolladores, sino de cualquier parte interesada del negocio afectada por cualquier fase del SDLC.

Para los desarrolladores, esto podría incluir la capacidad de aprovisionar infraestructura, compilar, empaquetar en contenedores y lanzar su software con facilidad, así como depurar incidencias sencillamente cuando algo falla. Para los SRE, incluye requisitos multifuncionales como observabilidad, composabilidad, reemplazabilidad, confiabilidad, escalabilidad, viabilidad y usabilidad. Los equipos de seguridad y cumplimiento se preocuparán por los marcos regulatorios y la privacidad del cliente. Existen muchas más partes interesadas, pero una de las más importantes a considerar desde el principio es el propio equipo de plataforma. 

En muchas organizaciones, los gerentes de producto (*product managers*) también son partes interesadas clave para la plataforma. Representan las necesidades de los desarrolladores, ayudan a priorizar capacidades y aseguran que el trabajo de la plataforma se alinee con los resultados del negocio. Por lo tanto, los ingenieros de plataforma y los gerentes de producto colaboran continuamente. Los requisitos se recopilan primero mediante entrevistas a desarrolladores, datos de uso y puntos de fricción operativa, documentándose luego como capacidades de la plataforma en lugar de tareas de infraestructura. Este cambio evita que el backlog de la plataforma se convierta en una simple colección de actualizaciones técnicas y lo mantiene enfocado en mejorar los flujos de trabajo de los desarrolladores y los resultados de entrega. El equipo de ingeniería de la plataforma debe disfrutar de la misma experiencia que cualquier otro equipo de desarrollo de la organización, y eso comienza facilitando el cumplimiento de las mejores prácticas de desarrollo.

##### Caminos Dorados y Valores por Defecto (*Golden Paths and Defaults*)
Los caminos dorados (*Golden Paths*) son los flujos de trabajo respaldados y basados en opiniones (*opinionated*) que ayudan a los desarrolladores a crear y enviar software de forma rápida y segura. Al establecer valores predeterminados como plantillas estándar de CI/CD, esquemas de infraestructura y enfoques de prueba recomendados, el equipo de plataforma reduce la fricción y la fatiga por toma de decisiones, manteniendo al mismo tiempo la flexibilidad para casos avanzados.

##### Bucles de Retroalimentación y Métricas (*Feedback Loops and Metrics*)
Una plataforma de ingeniería solo es valiosa si se adapta continuamente a las necesidades de los usuarios. Podríamos afirmar que este es el mayor desafío individual en la ingeniería de plataformas. Esto requiere bucles de retroalimentación cortos, donde los desarrolladores puedan señalar puntos de fricción y el equipo de plataforma pueda medir la adopción, la confiabilidad y la satisfacción mediante métricas como el tiempo de entrega (*lead time*), la tasa de fallos en cambios (*change failure rate*) y encuestas de experiencia del desarrollador. Estas métricas sirven como los criterios de éxito *de facto* para la plataforma.

> El equipo de María adopta la canalización de despliegue de la plataforma e inicialmente informa que la incorporación es fluida. Sin embargo, en pocas semanas el equipo de plataforma detecta un aumento en el tiempo de entrega y un repunte en las reejecuciones de trabajos de CI. Una breve sesión de retroalimentación revela que los desarrolladores esperan escaneos de seguridad que se ejecutan tarde en la canalización. El equipo de la plataforma adelanta el escaneo e introduce una comprobación de dependencias en caché. El tiempo de entrega disminuye, los reintentos de CI se reducen y el equipo de María reporta menos interrupciones. La plataforma no mejoró debido a un rediseño completo, sino porque el bucle de retroalimentación reveló dónde existía realmente la fricción.

#### Higiene de Equipo y Código (*Team and Code Hygiene*)

##### Repositorios Delimitados por Dominio (*Domain-Bounded Repositories*)
La mayoría de las aplicaciones modernas siguen prácticas de diseño guiado por el dominio (*Domain-Driven Design*), y una plataforma de ingeniería no debería ser diferente. Una plataforma debe diseñarse para la escalabilidad a medida que más y más equipos la adopten, por lo que debemos permitir que varios desarrolladores o incluso equipos trabajen en ella simultáneamente. Algunos ejemplos de dominios para una plataforma de ingeniería podrían ser:
- Habilitación de CI/CD
- Redes (*Networking*)
- Entorno de ejecución (*Runtime*)
- Aprovisionamiento de infraestructura de autoservicio
- Observabilidad

##### Prácticas de Equipo y Cultura (*Team Practices and Culture*)
Para alguien con experiencia en operaciones o DevOps, puede resultar sorprendente que la higiene de equipo sea uno de los conceptos clave a explorar con mayor profundidad. Sin embargo, dado que queremos desarrollar una plataforma igual que cualquier otro software, las prácticas fundamentales (como la estructura de confirmaciones, convenciones de etiquetas y propiedad del código), que inicialmente pueden parecer procedimentales, evolucionarán con el tiempo hacia hábitos de equipo y, en última instancia, cultura de ingeniería. Estas prácticas influyen no solo en la usabilidad y adopción de la plataforma, sino también en la mantenibilidad y extensibilidad a largo plazo.

##### CI para Plataformas (*CI For Platforms*)
Para lograr la máxima eficiencia, todos los aspectos de una plataforma de ingeniería deben construirse, configurarse y desplegarse mediante automatización. A medida que desarrollemos una plataforma en este libro, utilizaremos el desarrollo basado en la rama principal (*trunk-based development*), una práctica que recomendamos seguir también en otros proyectos. Siempre que se cuente con un código fuente correctamente delimitado por dominios, esta práctica resulta muy eficaz para minimizar conflictos de fusión, evitar desviaciones de infraestructura (*infrastructure drift*) y controlar costes mediante entornos mínimos y límites de seguridad (*guardrails*) consistentes.

Si bien la automatización centralizada mejora la consistencia, introduce ciertas concesiones. Las canalizaciones compartidas pueden reducir el aislamiento entre equipos, lo que requiere una validación más rigurosa y pruebas de contrato. A menudo, las banderas de características (*feature flags*) se vuelven necesarias para evolucionar las capacidades de la plataforma de forma segura sin bloquear a los equipos. Además, las canalizaciones de CI de la plataforma deben ser más robustas que las de las aplicaciones, ya que un fallo afecta a muchos equipos a la vez. El objetivo no es eliminar estos desafíos, sino hacerlos predecibles y manejables.

##### Pruebas (*Testing*)
A medida que las plataformas se vuelven más complejas con arquitecturas altamente componibles, establecer una pirámide de pruebas adecuada con los tipos de pruebas apropiados (como pruebas unitarias de infraestructura, entornos efímeros de integración y pruebas de contrato para las API de la plataforma), junto con los entornos en los que se aplican, es esencial para el éxito general. A medida que los equipos comiencen a adoptar la plataforma, necesitarán confiar en que los entornos a los que despliegan son estables. De este modo, cuando ocurran incidencias, los equipos tendrán la certeza de que se trata de un problema que deben solucionar en su código, y no de un fallo surgido en la propia plataforma tras un lanzamiento por el que tengan que esperar una corrección.

#### Seguridad y Cumplimiento (*Security and Compliance*)

##### Gestión de Secretos (*Managing Secrets*)
Gestionar los secretos de forma metódica y estructurada es más que una simple actividad de higiene para un equipo de plataforma; por ello lo destacamos como un concepto independiente. Esto debe formar parte del ADN de la filosofía de desarrollo del equipo para que las plataformas sean aceptadas en un marco empresarial. La razón es que los secretos serán utilizados tanto por la propia plataforma de ingeniería como por los equipos de desarrollo. Existe una alta expectativa de que el equipo de plataforma gestione esto con eficacia.

Aunque tratamos los secretos en esta sección, la arquitectura **Zero-Trust** (Confianza Cero) es un principio arquitectónico mucho más amplio. La gestión de secretos es solo un mecanismo de aplicación. Una plataforma que opera bajo Confianza Cero no asume ninguna confianza implícita entre usuarios, servicios, redes o infraestructura. La verificación de identidad, la autenticación de cargas de trabajo, las políticas de red, la rotación de certificados y la autorización de privilegios mínimos funcionan conjuntamente. Los secretos son simplemente el material de credenciales utilizado por estos controles, no el modelo de seguridad en sí mismo.

##### Principios de Plataforma Segura por Diseño (*Secure-by-Design Platform Principles*)
La seguridad no debe añadirse al final, sino diseñarse dentro de la plataforma desde el principio. Esto incluye aplicar el principio de mínimo privilegio, adoptar prácticas de confianza cero, garantizar el cumplimiento de los estándares organizacionales y hacer que los valores predeterminados seguros sean el camino más fácil para los desarrolladores. Al integrar barreras de seguridad en las canalizaciones y en la infraestructura, el equipo de plataforma ayuda a los desarrolladores a realizar entregas con mayor rapidez sin sacrificar el cumplimiento ni exponer riesgos.

Las organizaciones modernas confían cada vez más en las plataformas para operativizar el cumplimiento normativo en lugar de aplicarlo manualmente. Al incorporar políticas en las canalizaciones, controles de acceso y aprovisionamiento de infraestructura, la plataforma puede aplicar automáticamente estándares como **ISO 27001** o **SOC 2**, generando simultáneamente evidencias de auditoría. En lugar de que los desarrolladores rellenen formularios o recopilen capturas de pantalla, el cumplimiento se convierte en un subproducto de los flujos de trabajo habituales de ingeniería. Esto reduce la sobrecarga operativa y asegura que la adhesión sea continua en vez de periódica.

Más allá de hacer cumplir la seguridad, una plataforma debe hacer que su comportamiento sea observable y explicable. La auditabilidad garantiza que los equipos puedan responder quién cambió qué, cuándo y por qué, sin tener que reconstruir el historial manualmente. Esto incluye registros inmutables, trazabilidad de despliegues, registros de cambios de infraestructura y decisiones de políticas registradas automáticamente. En industrias reguladas, esto respalda el cumplimiento; pero incluso fuera de ellas genera confianza, ya que los desarrolladores siempre adoptan plataformas sobre las que pueden razonar y depurar con seguridad.

Las plataformas también acumulan deuda técnica, aunque de manera diferente a las aplicaciones. Pequeñas inconsistencias (como plantillas duplicadas, canalizaciones para casos especiales o excepciones a las barreras de seguridad) se convierten gradualmente en riesgos operativos porque afectan a muchos equipos simultáneamente. Por ello, los equipos de plataforma deben gestionar la deuda de forma intencionada mediante políticas de obsolescencia (*deprecation*), capacidades versionadas y rutas de migración claras. El objetivo no es evitar el cambio, sino evolucionar de forma segura sin imponer reescrituras disruptivas en toda la organización.

---

### Una Breve Discusión sobre el Stack de Herramientas: ¿Por Qué Estas Herramientas?

Aunque normalmente evitamos basar las discusiones de ingeniería de plataformas exclusivamente en herramientas, la naturaleza de este libro requiere asegurarnos de que conozcas y estés preparado para utilizar los componentes esenciales de la pila tecnológica que construiremos a lo largo del texto. La Figura 1.2 describe los elementos críticos de nuestro stack.

> **Figura 1.2** - Panorama de herramientas en el contexto de este libro

Las herramientas clave seleccionadas para este libro se presentan en la Figura 1.2. Los criterios de selección se basaron en las siguientes preguntas:
- ¿Pueden los lectores experimentar y aprender practicando sin una inversión adicional significativa en licencias?
- ¿Son herramientas probadas y contrastadas en entornos reales de ingeniería?
- ¿Quedarán los lectores atados a un único proveedor o estas herramientas ofrecen opciones portables e intercambiables?
- Utilizarás estas herramientas como parte de un sistema global: ¿funcionan bien juntas para demostrar la automatización de extremo a extremo en el contexto de una plataforma?

Repasemos brevemente cada componente del panorama:
- **Infraestructura como Código (IaC)**: Pulumi con Python y UV proporciona una pila de IaC consistente, comprobable y fácil de administrar, a la vez que se alinea con las mejores prácticas de desarrollo de software.
- **Gestión de Secretos**: Bitwarden permite el almacenamiento y la recuperación segura de secretos accesible mediante CLI, de modo que ninguna credencial resida en el código ni en archivos estáticos.
- **Control de Código Fuente**: GitHub ofrece una capacidad de gestión de repositorios sólida e inigualable, al tiempo que integra controles de políticas y un ecosistema global más amplio.
- **CI/CD**: El nivel gratuito de CircleCI admite canalizaciones automatizadas, configuraciones reutilizables y ejecutores locales (*local runners*) para despliegues de plataforma totalmente autocontenidos.
- **Orquestación de Contenedores**: Kind y Helm ofrecen clústeres locales de Kubernetes y despliegues basados en plantillas para un flujo de trabajo de orquestación similar al de producción.
- **Calidad de Código**: Un conjunto de herramientas de Python como `pytest`, `ruff` (linting, formateo y organización de importaciones), `mypy` y `bandit` garantiza pruebas, consistencia de código, seguridad de tipos e higiene de seguridad.

Si deseas seguir el desarrollo y despliegue de la plataforma de ingeniería de este libro, consulta el Apéndice A para ver una guía paso a paso sobre la configuración de tu entorno de desarrollo local con todas las herramientas necesarias.

Para concluir esta sección, observemos un flujo de datos potencial en la plataforma de ingeniería que estás desarrollando:
1. El desarrollador envía el código de infraestructura de Pulumi a GitHub.
2. El webhook de GitHub activa el ejecutor de CircleCI que se ejecuta en una máquina local.
3. El ejecutor de CircleCI utiliza la CLI de Bitwarden para obtener las credenciales y secretos necesarios.
4. Luego, Pulumi previsualiza, valida los cambios y despliega la infraestructura.
5. Los gráficos (*charts*) de Helm se despliegan en la nueva infraestructura y las cargas de trabajo gestionadas comienzan a ejecutarse.

Utilizamos Pulumi para demostrar la infraestructura como software en lugar de infraestructura como configuración. Un lenguaje de propósito general permite aplicar al código de plataforma las mismas prácticas de pruebas, empaquetado, validación y revisión de código utilizadas para el desarrollo de aplicaciones. Esto refuerza el modelo de plataforma como producto, donde la infraestructura evoluciona a través de los flujos de trabajo de ingeniería habituales. Los patrones descritos aquí se trasladan directamente a Terraform u otras herramientas de IaC. El enfoque reside en el flujo de trabajo y el ciclo de vida, no en la tecnología de implementación específica.

---

### Topología del Repositorio de Código Fuente

El primer paso para crear una plataforma de ingeniería es crear un repositorio de código fuente. Como se mencionó en la sección anterior, deseamos alinear la estructura de nuestro código fuente con los dominios de la plataforma. Para garantizar la escalabilidad futura y un mantenimiento eficiente, resulta beneficioso crear repositorios de código fuente independientes (o al menos canalizaciones de despliegue separadas) para los distintos dominios. A medida que construyas tu plataforma de ingeniería junto a nosotros a lo largo de este libro, reflexiona sobre los pros y los contras de tener un monorepositorio (*monorepo*) frente a múltiples repositorios focalizados (*polyrepo*). Aunque la comparativa exhaustiva de estos enfoques queda fuera del alcance de este libro, te animamos a profundizar en sus ventajas y desventajas. Según el flujo de trabajo de tu equipo y las capacidades proyectadas de la plataforma, esta decisión influirá notablemente en la seguridad de la integración, la velocidad de lanzamiento y el riesgo operativo.

Aunque la estructura de repositorios y el flujo de trabajo de desarrollo son técnicamente decisiones independientes, en la práctica se influyen mutuamente. El desarrollo basado en la rama principal (*trunk-based development*) funciona de manera natural con monorrepositorios porque los cambios en dependencias se vuelven visibles de inmediato y la integración ocurre de forma continua. En configuraciones polirrepositorio, los flujos basados en la rama principal requieren una disciplina de versionado más estricta y pruebas de contrato para evitar cambios incompatibles ocultos entre servicios. Por tanto, los equipos de plataforma deben elegir la combinación de forma consciente: el monorrepositorio optimiza la velocidad y la consistencia de la integración, mientras que el polirrepositorio optimiza el aislamiento y la autonomía.

#### Dominios de la Plataforma

Un aspecto clave del enfoque de plataforma que adoptaremos es la comprensión inherente de los límites del dominio. Aunque no entraremos en los detalles específicos de cada uno, identificar los puntos naturales de separación en tu plataforma es fundamental. Los dominios en este contexto no son los dominios de negocio para los cuales se construye el producto, sino más bien los dominios de la propia plataforma. Por lo general, se dividen en ocho categorías diferentes, agrupadas convenientemente en la capa de experiencia de usuario de la plataforma, la capa de aplicación de usuario de la plataforma y la capa fundacional. Para obtener más información al respecto, sugerimos a los usuarios consultar el trabajo original donde introdujimos este concepto [1]. Al trabajar en el marco de tu repositorio de código fuente, recuerda dividirlo a lo largo de los tres ejes mostrados a continuación. En la Figura 1.3 puedes observar la estructura organizativa de alto nivel según los límites de dominio.

> **Figura 1.3** - Vista de alto nivel de los límites de dominio para el código de la plataforma

Los dominios en el diagrama representan límites de responsabilidad de la plataforma en lugar de equipos organizativos:
- **Capa de Usuario de la Plataforma** (*Platform User Layer*): puntos de entrada a través de los cuales los desarrolladores interactúan con la plataforma (CLI, portal, integraciones de IDE).
- **Dominios de la Capa de Experiencia** (*Experience Layer Domains*): servicios orientados al desarrollador como identidad, incorporación y capacidades de la plataforma.
- **Dominios de la Capa de Aplicación** (*Application Layer Domains*): lógica compartida del plano de control y mecanismos de extensibilidad utilizados por las cargas de trabajo.
- **Dominios Fundacionales** (*Foundational Domains*): capacidades base en la nube como redes, estructura de cuentas e identidad administrativa.

La seguridad y el cumplimiento abarcan todos los dominios. En lugar de tratarse como un único componente, se aplican a través de barreras de seguridad, controles de políticas y mecanismos de auditoría integrados en todas las capas.

Para este libro, utilizaremos repositorios diferentes, cada uno con su propia canalización, para los dominios de la plataforma, imitando una iniciativa a gran escala que permite a varios equipos trabajar en paralelo en distintas partes de la plataforma. Como nuevo equipo de plataforma, es posible que tu backlog aún no tenga cada dominio perfectamente definido, lo que nos permite comenzar rápidamente y realizar entregas incrementales. Además de esto, también queremos asegurarnos de que cada repositorio se configure de la misma manera. Esto puede incluir:
- Añadir miembros del equipo de plataforma al repositorio con acceso de "Admin" o "Member".
- Convenciones de nomenclatura para repositorios y ramas.
- Políticas para el repositorio, como requisitos de firma de código, políticas de pull requests y más.

Para simplificar las cosas, con el espíritu de "automatizarlo todo", ¡debemos automatizar la creación de nuestros repositorios! De esa manera, garantizaremos una configuración consistente y facilitaremos una escalabilidad sencilla a medida que se identifiquen nuevos dominios.

#### Configuración de la Gestión de Secretos del Equipo

El primer repositorio que debe tener nuestro equipo de plataforma es uno destinado a gestionar lo que el equipo necesita para operar, siendo lo más importante los repositorios delimitados por dominio. Podemos gestionar los artefactos de GitHub con Infraestructura como Código del mismo modo que gestionaríamos recursos en un proveedor de nube. Sin embargo, no queremos empezar a crear scripts sin ninguna estructura. Deseamos establecer rigor en nuestras prácticas de desarrollo lo antes posible, asegurando que todo lo realizado en una máquina local sea repetible por cualquier miembro del equipo de desarrollo para poder distribuir el trabajo a lo largo del tiempo.

En primer lugar, queremos crear una estructura de carpetas para un proyecto de Pulumi en Python, de modo que cualquier persona que lo utilice pueda localizar fácilmente la parte específica del módulo en la que necesita trabajar. Crea una carpeta para el módulo llamada `platform-team-administration` y dentro de ella ejecuta `pulumi new python`. Puedes aceptar los valores predeterminados en la mayoría de las opciones, pero asegúrate de seleccionar `uv` para la gestión de paquetes si estás siguiendo el proceso. A continuación, además de los archivos y carpetas creados por Pulumi, crea la siguiente estructura de carpetas:

> **Figura 1.4** - Estructura inicial de carpetas del repositorio

A continuación, queremos configurar nuestro almacén de secretos para guardar los tokens de acceso de modo que puedan utilizarse en los procesos de CI/CD, pero asegurándonos de que los secretos no se incluyan en nuestro repositorio git. Crearemos dos archivos: uno almacenado localmente con los secretos y otro que sí podamos confirmar en el repositorio, el cual mostrará a otros miembros del equipo cómo configurarlo en su máquina local. En primer lugar, crea un archivo llamado `.env_example` en la raíz del directorio con el siguiente código:

```bash
BW_CLIENTID=your_bitwarden_api_key_client_id
BW_CLIENTSECRET=your_bitwarden__api_key_client_secret
BW_PASSWORD=your_bitwarden_master_password
```

Duplica este archivo en uno nuevo llamado `.env` e introduce en los marcadores de posición la información de Bitwarden que registraste al configurar el conjunto de herramientas de tu organización. Por último, crea un archivo `.gitignore` en la raíz del directorio con la entrada `.env` para evitar que se suba al control de versiones. Ahora estamos listos para almacenar nuestros secretos de tokens.

Para ello, seguiremos el mismo patrón de crear un archivo de ejemplo que pueda incluirse en el control de versiones junto a una copia local. De ese modo, estaremos preparados para actualizar o introducir nuevos secretos fácilmente en el futuro. Crearemos un archivo en el formato que Bitwarden espera cuando se inyecta un secreto mediante la CLI `bw`. En la carpeta `secrets-setup`, crea un archivo llamado `github_secrets.json_example`, junto con `github_secrets.json` que contendrá los valores reales. No olvides añadir una entrada en `.gitignore` para el archivo con los valores reales:

```json
{
  "type": 1,
  "name": "GitHub Secrets",
  "notes": "Bitwarden Secrets for Pulumi GitHub provider",
  "fields": [
    {
      "name": "pulumi-github-token",
      "value": "your_github_pat_token",
      "type": 0
    },
    {
      "name": "pulumi-github-owner",
      "value": "your_github_organization_name",
      "type": 0
    }
  ],
  "login": {
    "uris": [],
    "username": null,
    "password": null,
    "totp": null,
    "passwordRevisionDate": null
  }
}
```

El campo `type` especifica la categoría del elemento de Bitwarden. `type: 1` representa un elemento de inicio de sesión (*Login*), necesario porque la CLI de Bitwarden recupera credenciales de secretos de tipo login. Por lo tanto, los tokens y las claves API se almacenan como elementos de inicio de sesión en lugar de notas seguras.

¿Qué valores debes introducir?
- `username` → déjalo como `null` (no se requiere para el uso de tokens de GitHub)
- `password` → pega tu Token de Acceso Personal (PAT) de GitHub
- `totp` → déjalo como `null`
- `uris` → opcional (puedes establecer `https://github.com`, pero no es obligatorio)
- `passwordRevisionDate` → déjalo como `null` (Bitwarden lo establece automáticamente)

Para este ejemplo, solo el campo `password` debe contener un valor. El resto de campos existen porque forman parte del esquema estándar de Bitwarden.

Ahora podemos crear un script para inyectar estos secretos en nuestro almacén de Bitwarden utilizando la CLI. En `secrets-setup`, crea un archivo llamado `inject_secrets.sh` que pueda iterar sobre cualquier plantilla JSON en la carpeta y crear las entradas, y asegúrate de que sea ejecutable ejecutando `chmod +x inject_secrets.sh`.

En primer lugar, asegúrate de que las variables de entorno para Bitwarden estén establecidas y obtén una sesión de bóveda:

```bash
set -o allexport
source "../.env"
set +o allexport

bw login --apikey
BW_SESSION=$(bw unlock --passwordenv BW_PASSWORD --raw)
```

A continuación, queremos iterar sobre todos los archivos JSON del directorio y cargarlos en Bitwarden. Primero comprobaremos si el secreto ya existe para garantizar que se actualice o se cree según corresponda:

```bash
for json_file in *.json; do
  echo "Processing: $json_file"
  # Extract item name from filename (remove .json extension)
  item_name=$(cat $json_file | jq -r .name)
  # Try to get an existing item, if it exists, update, otherwise
  # create new
  existing_item=$(bw get item "$item_name" \
    --session "$BW_SESSION" 2>/dev/null)
  if [ -n "$existing_item" ]; then
    echo "Item '$item_name' exists, updating..."
    item_id=$(echo "$existing_item" | jq -r .id)
    cat "$json_file" | bw encode | bw edit item "$item_id" \
      --session "$BW_SESSION"
  else
    echo "Item '$item_name' does not exist, creating..."
    cat "$json_file" | bw encode | bw create item --session "$BW_SESSION"
  fi
done
```

Por último, sincroniza la información con la nube de Bitwarden y bloquea la bóveda para que la clave de sesión quede invalidada. Esto garantizará que otros miembros del equipo puedan acceder a los secretos en el futuro si se incorporan a la organización:

```bash
bw sync --session "$BW_SESSION"
bw lock
```

Ahora, si inicias sesión en tu cuenta en [bitwarden.com](https://bitwarden.com/), ¡deberías ver tus secretos cargados!

Una vez cargados en la bóveda de secretos los datos necesarios para iniciar sesión en nuestras organizaciones de GitHub y Pulumi, podemos crear los repositorios delimitados por dominio que utilizará nuestro equipo.

> **Ejercicio: Crear un archivo de plantilla para la clave API de Pulumi**
> 
> Crea un archivo JSON de plantilla en el formato de la CLI de Bitwarden y almacena un secreto con un campo que contenga tu clave API de Pulumi utilizando los siguientes valores:
> - **Nombre del secreto**: `"Pulumi Secrets"`
> - **Nombre del campo**: `"pulumi-api-key"`
> 
> *La solución a este ejercicio se encuentra en el repositorio complementario.*

#### Automatización de la Creación de Repositorios

Al utilizar IaC para crear repositorios Git, nuestro objetivo es crear una automatización que sirva como fuente central de verdad para los estándares del equipo sobre el uso del control de código fuente. Si simplemente quisiéramos crear un repositorio, no necesitaríamos IaC, ya que un repositorio no suele ser algo que se cree y destruya iterativamente. Al automatizarlo, una rutina centralizada y consistente garantizará que todos los repositorios sigan las mismas convenciones de configuración y acceso para el equipo de plataforma. Primero, debemos crear un flujo de trabajo de lo que esperamos que realice la automatización. Actualmente, queremos que `platform-team-administration` haga tres cosas:

> **Figura 1.5** - Automatización de repositorios

Para mantener esta automatización extensible, crea `config/platform_team_values.yaml` para cargar la configuración de cada paso.

Este archivo reside dentro del directorio de tu proyecto Pulumi, en el mismo lugar donde se encuentra el script de creación de repositorios de Pulumi. De este modo, a medida que sea necesario añadir repositorios o realizar cambios de gobernanza, tendremos una única fuente de verdad para lo que se gestiona. Por ahora, podemos crear tres repositorios:

```yaml
github_repositories:
  # Chapter 1 Repos
  - name: platform-team-admin
    description: Repository to manage platform team membership and admin artifacts
  - name: platform-core
    description: Core platform runtime
  - name: platform-demo-apps
    description: Demo application for testing the platform
```

Con la configuración lista, ahora podemos usar el proveedor GitHub de Pulumi para crear cada repositorio. Primero, necesitamos autorizar a nuestro IaC para acceder a GitHub utilizando las credenciales almacenadas en Bitwarden. Podemos recuperarlas con el proveedor de Bitwarden. Pulumi aún no dispone de un proveedor nativo de Bitwarden. Sin embargo, Pulumi puede reutilizar proveedores existentes de Terraform mediante su puente de proveedores (*provider bridge*). Por lo tanto, utilizaremos el proveedor de Terraform de Bitwarden a través de Pulumi. Seguimos escribiendo código de Pulumi, pero aprovechamos una implementación de proveedor existente en lugar de construir una desde cero.

Tras la instalación, los secretos pueden incorporarse al entorno de Pulumi para autenticar el proveedor de GitHub utilizando el archivo `.env` creado anteriormente, y podemos usar esto para enviar nuestros repositorios a GitHub.

**`pulumi_repo_create.py`**

```python
import yaml
import pulumi
from pulumi import ResourceOptions, export
from pulumi_github import Provider, Repository
```

Configurar el proveedor (nombre explícito y token provenientes de secretos):

```python
github_provider = Provider(
    "platform-github-provider",
    token=pulumi.Config("github").require("token"),
    owner=pulumi.Config("github").require("owner"),
)
```

Cargar definiciones de repositorios (archivo de configuración: `config/platform_team_values.yaml`):

```python
with open("config/platform_team_values.yaml") as f:
    data = yaml.safe_load(f)

for repo_def in data.get("github_repositories", []):
    repo_name = repo_def.get("name")
    repo_description = repo_def.get("description", "")
    # allow per-repo visibility: 'private' | 'public' | 'internal'
    visibility = repo_def.get("visibility", "private")
```

Crear el repositorio utilizando explícitamente el proveedor configurado:

```python
repository = Repository(
    repo_name,
    name=repo_name,
    description=repo_description,
    visibility=visibility,
    opts=ResourceOptions(provider=github_provider),
)
```

Exportar si otras pilas o ejemplos dependen del nombre:

```python
export(f"{repo_name}_repo_name", repository.name)
```

Ahora, asegúrate de que el código funciona ejecutando `pulumi preview` desde la CLI. Suponiendo que todo vaya bien, ¡ya podemos crear los repositorios con `pulumi up` y conectar este código inicial para confirmarlo! Si no estás seguro de cómo inicializar un repositorio git local y conectarlo a un repositorio remoto existente en GitHub, consulta la documentación de GitHub.

En adelante, ahora que puedes añadir rápidamente repositorios a medida que se identifiquen, debes considerar el manejo de librerías compartidas y dependencias entre repositorios al establecer la estructura. De esta manera, podrás asegurarte de exportar los valores clave para que otros espacios de trabajo de IaC los utilicen (como nombres de repositorios y otros metadatos). Aunque no lo implementaremos en detalle aquí, estas son las consideraciones principales a tener en cuenta:

- Decidir si necesitas configurar un monorrepositorio o un polirrepositorio.
- Asegurar que cada repositorio o carpeta tenga un propósito de dominio claro identificado por el concepto de dominios de la plataforma (por ejemplo, nombrar los repositorios como `platform-networking`, `platform-identity`, etc.).
- Prestar atención a cómo reducir el radio de impacto (*blast radius*) de cualquier cambio gestionando las dependencias adecuadamente, tanto de forma interna en tu repositorio como externamente.
- Asegurar que dispones de un repositorio guiado por configuración, a través del cual puedas migrar de un enfoque a otro sin tener que realizar refactorizaciones significativas ni romper los flujos de trabajo existentes (por ejemplo, si no introduces una capacidad de IA agéntica al principio pero deseas incorporarla más adelante, esto no debería requerir cambios a gran escala en tu arquitectura).

> **Ejercicio: Asegurarse de que los repositorios no se puedan eliminar**
> 
> Con la configuración actual desplegada, los repositorios que hemos comenzado a administrar podrían eliminarse ejecutando `pulumi destroy` desde la CLI. Este ciertamente no es un resultado deseado, ya que podría ocasionar una gran pérdida para el equipo de desarrollo. Investiga cómo garantizar la protección contra eliminación en los recursos de Pulumi e integra esta característica en tu código. Una vez completado, declara un nuevo repositorio de equipo llamado `platform-extensions` y despliégalo en GitHub.

#### Automatización de la Incorporación del Equipo de Plataforma

En una gran organización, los miembros de un equipo de ingeniería de plataformas suelen unirse y abandonar el equipo con regularidad, ya sea por rotación natural, reestructuración organizativa o cambios en las necesidades de capacidad. Como parte de la automatización de la administración del equipo de plataforma, habilitar la incorporación (*onboarding*) y desvinculación (*offboarding*) de miembros en "1 clic" resulta muy útil. Esto se puede lograr incluyendo la capacidad de añadir y eliminar miembros de conjuntos de herramientas, bóvedas u otros recursos utilizando la misma configuración empleada para crear nuestros repositorios. De las herramientas configuradas hasta ahora, Bitwarden tiene limitaciones en su API que impiden la automatización de usuarios, pero sí podemos controlar el acceso a GitHub de esta forma.

Antes de comenzar, necesitarás añadir un segundo usuario de GitHub bajo tu control para simular a alguien nuevo que se une a tu equipo de plataforma, ya que no se puede automatizar la gestión del propietario de una organización. Si utilizas Gmail, puedes crear una nueva cuenta de GitHub usando un alias con el formato `tucorreo+peh-team-member@gmail.com`, lo que enviará todas las verificaciones a tu dirección de correo habitual. Una vez que tengas un miembro de equipo adicional, se puede añadir una nueva entrada de configuración en `config/platform_team_values.yaml`:

```yaml
github_organization_members:
  - name: Team Member
    github-role: admin
    github-username: team_member
    email: team_member@email.com
```

A continuación, en `pulumi_repo_create.py`, justo antes de que se creen los repositorios de GitHub, añade a los miembros del equipo y sus roles asociados a partir del archivo de configuración:

```python
for team_member in data.get("github_organization_members", []):
    name = team_member.get("name")
    username = team_member.get("github-username")
    role = team_member.get("github-role", "member")

    for member in data["github_organization_members"]:
        username = member["github-username"]
        role = member["github-role"]
        team_membership = github.Membership(
            f"github-membership-{username}",
            username=username,
            role=role,
        )
```

Una vez que apliques este código con `pulumi up`, deberías ver una invitación a la organización enviada a tu correo electrónico para el nuevo miembro del equipo de plataforma, lo que da como resultado una rutina de incorporación rápida y sencilla. El acceso administrativo debe concederse a una cuenta de servicio de la plataforma o a un grupo del equipo de la plataforma, no a ingenieros individuales.

> **Ejercicio: Dar de baja a un miembro del equipo (*Offboarding*)**
> 
> Experimenta eliminando a tu miembro del equipo en el código de IaC para asegurarte de que ya no pueda acceder a los repositorios de la organización una vez aplicado el código. Asimismo, siéntete libre de experimentar cambiando el rol en la organización de `admin` a `member` y observa qué diferencias surgen en los recursos a los que puede acceder el nuevo miembro.

---

### Garantizar las Convenciones de Confirmación (*Commit Conventions*)

Una vez tomadas las decisiones de configuración del repositorio, es necesario definir las convenciones específicas que utilizarás para confirmar el código. Comienza definiendo la estructura de las confirmaciones (por ejemplo, qué información se requiere en el mensaje de confirmación) y luego incluye los tipos de confirmación: si se trata de una nueva característica, corrección de errores, documentación, refactorización, pruebas, etc. Además, etiqueta las confirmaciones para asegurarte de poder encontrarlas fácilmente más adelante, independientemente de la herramienta de seguimiento que utilices. Una gran ventaja es la posibilidad de configurar enlaces de pre-confirmación (*pre-commit hooks*) para validar el mensaje de confirmación. Debes utilizar cualquier herramienta que ya emplees para actividades interactivas relacionadas, así como para análisis estático (*linting*) y ejecución de pruebas rápidas que las validen. Cuantos más desarrolladores contribuyan al código de la plataforma, más podrás hacer que los hooks sean una parte obligatoria de su flujo de trabajo.

Las convenciones de confirmación y la automatización de políticas no son simples reglas arbitrarias: son fundamentales para una experiencia fluida del desarrollador y esenciales para escalar el cumplimiento en organizaciones más grandes. Al alinear a los equipos desde el principio, puedes transformar futuros cuellos de botella en flujos de trabajo predecibles que respalden tanto la velocidad de los desarrolladores como la confianza general de la organización.

Para barreras de seguridad adicionales, considera hacerlas tan rigurosas o flexibles como estimes conveniente. Recomendamos basarte en tu experiencia general gestionando repositorios de código complejos. Recuerda que estás tratando a tu plataforma igual que a cualquier otro producto de software de la organización, por lo que el código que desarrollaremos tendrá un ciclo de vida prolongado; a menudo, una vida útil más larga que el código de producto visible para los clientes. Generar registros de cambios (*changelogs*) a partir del historial de confirmaciones es crucial, y esto también debería servir para incrementar los números de versión automáticamente. La automatización es clave aquí, ya que no deseas encontrarte en un caos de versiones a los dos meses de comenzar el desarrollo de la plataforma. La automatización también debe extenderse a las actividades de alerta a las partes interesadas y generación de informes adecuados. Si bien no recomendamos cambiar tus herramientas actuales, soluciones como GitLab (con licencia) o herramientas de código abierto como Release Please o Semantic Release pueden gestionar la mayoría de estas tareas.

#### Habilitar al Equipo de Plataforma para las Convenciones del Repositorio

El primer paso para habilitar al equipo de plataforma en las convenciones establecidas para el repositorio consiste en identificar los detalles esenciales sobre el tipo, el alcance y la descripción. También debes definir tipos de confirmación para diferenciar los cambios (características, correcciones de errores, pruebas o cualquier otro tipo de cambio). Luego deberás etiquetar tus confirmaciones para localizar las modificaciones fácilmente más adelante.

Una vez establecidos los aspectos esenciales mencionados, considera configurar hooks de pre-commit para validar los mensajes y ejecutar pruebas de linting. Aunque muchos equipos de plataforma no comienzan configurando las barreras de seguridad de forma estricta, recomendamos encarecidamente hacerlo desde el primer momento. Las barreras pueden incluir reglas específicas sobre los changelogs, cómo incrementar las versiones automáticamente o integrar automatizaciones para notificar a los interesados y registrar eventos en la plataforma de observabilidad para auditorías.

No existe una forma directa de forzar a un miembro del equipo a instalar hooks de pre-commit en su máquina local. Aun así, puedes simplificar el proceso incluyendo las definiciones de los hooks en el repositorio de código fuente junto con un script que haga que la instalación sea fluida. Por ejemplo, si queremos seguir una convención de confirmación de código, crea un archivo llamado `commit-msg` en la carpeta `.git-hooks` de la siguiente manera:

```bash
#!/bin/bash
# Get the commit message
commit_msg=$(cat $1)

# Define a regex pattern to match the conventional commit message format
pattern='^(build|chore|ci|docs|feat|fix|perf|refactor|revert|style|test)(\([a-z]+\))?!?: .+$'

# Test if the commit message matches the pattern
if ! [[ $commit_msg =~ $pattern ]]; then
    echo "ERROR: The commit message does not match the Conventional Commits format."
    echo "  Please use the format: type(scope)?: subject"
    echo "  Where 'type' is one of: build, chore, ci, docs, feat, fix, perf, refactor, revert, style, test."
    echo "  Example: feat(users): add new user endpoint"
    echo "  See https://www.conventionalcommits.org/en/v1.0.0/#summary for more information"
    exit 1
fi

# If the commit message matches the pattern, exit successfully
exit 0
```

A continuación, en la carpeta `scripts`, crea un archivo llamado `install-githooks.sh` que instalará este archivo en la carpeta estándar de hooks de Git de la máquina local:

```bash
cp .git-hooks/commit-msg .git/hooks/commit-msg
```

En el código complementario disponemos de hooks adicionales definidos, como `pre-commit` para actividades como linting de código, verificación de formato y otras. Siéntete libre de adoptarlos o experimentar con más revisando los hooks disponibles en el directorio `.git/hooks` del proyecto.

#### Automatización de la Aplicación de Políticas del Repositorio

A medida que implementes herramientas o scripts para aplicar automáticamente las políticas del repositorio, acuerda primero los estándares con tus equipos. Estos estándares pueden referirse a reglas de protección de ramas o requisitos de solicitudes de fusión (*merge requests*). Dado que en este caso utilizamos GitHub, integrarlo estrechamente para evitar la fusión de confirmaciones no conformes será una necesidad absoluta. Luego considerarás extender la automatización para gestionar alertas y reportes de cumplimiento requeridos para la gobernanza.

GitHub ofrece la capacidad de definir políticas para exigir estándares de confirmación. Una en particular, fundamental para la gobernanza en industrias reguladas, es la **firma de confirmaciones de código** (*code commit signing*). Mediante esta práctica, cada confirmación debe estar firmada criptográficamente con un certificado SSH o GPG creado por cada desarrollador individual.

A continuación, en nuestro código de IaC, podemos aplicar una política para las confirmaciones en la rama `main` de cada repositorio creado, definiendo un recurso en el bucle de creación de `pulumi_repo_create.py` justo después de que se cree un repositorio:

```python
# Create a branch protection rule that enforces signed commits
branch_protection = github.BranchProtection(
    f"{repo_name}-main-branch-protection",
    repository_id=repo_name,
    pattern="main",
    enforce_admins=True,
    require_signed_commits=True,
)
```

Si aplicas este código con `pulumi up` e intentas confirmar cambios en GitHub, deberías recibir un error indicando que el código no se puede confirmar a menos que esté firmado. Para solucionar esto, crea un certificado y asócialo a tu cuenta de GitHub como certificado de firma de código siguiendo la documentación de GitHub en [https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits). Tan pronto como esté configurado, podrás confirmar tu código normalmente.

> **Ejercicio: Modificar la política del repositorio**
> 
> Modifica la política del repositorio para que se aplique a todas las ramas, no solo a `main`. De esta manera, podrás asegurar que cualquier confirmación pueda rastrearse hasta un desarrollador autorizado para cumplir con cualquier requisito de auditoría organizativa.

---

### Estrategia de Ramas y Etiquetado de Lanzamiento (*Branching and Release Tagging Strategy*)

Con los repositorios iniciales creados y las barreras de seguridad aplicadas, el equipo de plataforma está listo para comenzar a confirmar código en serio para construir la plataforma.

Con la estructura del repositorio, las convenciones de confirmación y la automatización de políticas implementadas, esta sección reúne esas piezas en una estrategia de lanzamiento optimizada, convirtiendo los principios de DevEx en confiabilidad de entrega diaria.

Antes de que esto ocurra, debemos establecer procesos de lanzamiento automatizados y ejecutados mediante CI/CD en cada operación de confirmación, fusión y etiquetado. Para garantizar la mantenibilidad a largo plazo, esto requerirá una estrategia bien definida para ramificar y etiquetar el código. Aunque puedas preferir un enfoque específico de ramificación y etiquetado, en nuestra experiencia los equipos de plataforma funcionan mejor utilizando el **desarrollo basado en la rama principal** (*trunk-based development*) [2]. Esto significa que cada confirmación irá a la rama `main` y activará un nuevo despliegue en el entorno de desarrollo del equipo. Hemos comprobado que intentar utilizar ramas de características (*feature branches*) crea problemas significativos a gran escala, porque los cambios de infraestructura son más complejos de fusionar que los cambios de código de aplicación. Cuando todos tus ingenieros de plataforma confirman en la rama principal, se reducen los conflictos de fusión y se genera una retroalimentación más rápida. La organización también incurre en menos costes al no mantener infraestructura duplicada para cada desarrollador de la plataforma.

#### Confirmaciones por Push frente a Tag (*Push vs Tag Commits*)

Para gestionar de forma óptima la automatización de lanzamientos, recomendamos validar un despliegue concreto en las operaciones de `push` y publicarlo en las operaciones de `tag`. Recomendamos esto porque, a medida que la infraestructura de la plataforma se vuelve más compleja y el tiempo de actividad resulta cada vez más crítico con una mayor adopción, probablemente desearás revisar los cambios realizados en un despliegue determinado y aprobarlos. Al ejecutar pruebas automatizadas exhaustivas y validación de configuración durante el paso de vista previa (*preview*), esta revisión y aprobación en el paso de lanzamiento se vuelve menos pesada y consume menos tiempo.

Sin embargo, por muy cuidadoso que seas, no podrás evitar reversiones (*rollbacks*) en situaciones críticas (en la mayoría de los casos puede que ni siquiera sea culpa tuya). Para facilitarlo, considera construir activadores de reversión automatizados basados en diversas comprobaciones de estado y monitorización. En particular, las reversiones de migraciones de bases de datos suelen tener un impacto mayor y deben ser parte integral de los cambios de infraestructura. Establecer un mecanismo de reversión también implica contar con un calendario regular de pruebas de reversión.

Para crear una canalización de lanzamiento automatizada que se ejecute en `push` frente a `tag`, utilizaremos CircleCI configurado con un ejecutor local que se active con cada confirmación en nuestro repositorio. Primero, podemos configurar una canalización de CircleCI en nuestro repositorio agregando `.circleci/config.yml`. Si no estás familiarizado con él, CircleCI utiliza *Orbs* como mecanismo para código reutilizable, por lo que podemos usar los orbs de Pulumi y Python en la parte superior del archivo:

```yaml
version: 2.1
orbs:
  pulumi: pulumi/pulumi@2.1.0
  python: circleci/python@3.1.0
```

A continuación, definimos filtros que nos permitirán declarar qué comando debe ejecutarse en operaciones de `push` frente a `tag`:

```yaml
on-push-main: &on-push-main
  branches:
    only: /main/
  tags:
    ignore: /.*/
on-tag-main: &on-tag-main
  branches:
    ignore: /.*/
  tags:
    only: /.*/
```

¡Y eso es todo! Ya estamos listos para añadir flujos de trabajo para previsualizar y publicar nuestros cambios de código.

#### Configuración de Flujos de Trabajo de Lanzamiento (*Configuring Release Workflows*)

Ahora que estás listo para automatizar tus canalizaciones de lanzamiento, automatizarás el proceso desde la creación de la etiqueta hasta la publicación del artefacto. Para seguir este ejercicio, debes configurar un ejecutor local de CircleCI como se describe en el Apéndice A. Con eso en marcha, podemos crear dos flujos de trabajo en nuestra canalización de CircleCI: uno para `push` y otro para `tag`. Idealmente, deberían ser idénticos, excepto que el flujo de `tag` debe incluir el paso adicional de autorización y creación de infraestructura:

```yaml
workflows:
  preview:
    jobs:
      - pulumi-preview:
          context: *context
          pulumi_stack: dev
          filters: *on-push-main
  update:
    jobs:
      - pulumi-preview:
          context: *context
          pulumi_stack: dev
          filters: *on-tag-main
      - approvegithubchanges:
          type: approval
          requires:
            - pulumi-preview
          filters: *on-tag-main
      - pulumi-update:
          context: *context
          pulumi_stack: dev
          requires:
            - approvegithubchanges
          filters: *on-tag-main
```

Observa la inclusión de un `context` aquí. Así es como CircleCI hace referencia a los valores secretos para la canalización. En CircleCI, puedes crear un contexto llamado `PLATFORM_ADMIN` utilizando los valores del archivo `.env` para Bitwarden y referenciarlo en la parte superior del archivo para que los trabajos de Pulumi puedan usarlo:

```yaml
globals:
  - &context PLATFORM_ADMIN
```

Todo lo que queda ahora es definir los trabajos que espera el flujo de trabajo. Este es el momento adecuado para incluir pruebas automatizadas antes de los despliegues para detectar cualquier problema de forma temprana. Estos son algunos pasos recomendados para una canalización de IaC robusta:

> **Figura 1.6** - Automatización de lanzamientos

Aunque queda fuera del alcance de este libro, recomendamos encarecidamente incorporar mecanismos de reversión automatizados activados por fallos según lo consideres oportuno. Implementar un mecanismo de reversión es una cosa, pero hemos observado que si no se prueban regularmente los procesos de reversión para la infraestructura y la base de datos, estos podrían resultar poco útiles con el tiempo.

> **Ejercicio: Aplicar una etiqueta de lanzamiento (*Release Tag*)**
> 
> En este ejercicio, aplicarás una etiqueta de lanzamiento a tu repositorio para marcar un punto estable en el desarrollo de tu plataforma de ingeniería:
> 
> 1. Asegúrate de que todos los cambios estén confirmados y tu rama esté al día.
> 2. Desde tu repositorio local, crea una etiqueta:
>    ```bash
>    git tag -a v0.1.0 -m "First stable release of platform setup"
>    ```
> 3. Envía la etiqueta a tu repositorio remoto:
>    ```bash
>    git push origin v0.1.0
>    ```
> 4. Verifica en GitHub que la etiqueta aparezca ahora en la sección **Releases**.

---

### Resumen

En este capítulo, te hemos ayudado a sentar las bases para construir una plataforma de ingeniería confiable y escalable. Específicamente, has combinado los aprendizajes de configurar una estructura de repositorio adecuada, convenciones de confirmación y automatización de políticas. También aprendiste a aplicar una etiqueta, un paso pequeño pero crucial que hace que tu proceso de desarrollo sea repetible y confiable. En conjunto, estas prácticas forman la primera capa para transformar los principios de DevEx en confiabilidad de entrega, conectando tu trabajo diario de desarrollo con la plataforma más amplia que estamos construyendo a lo largo del resto del libro.

De cara al futuro, en el [Capítulo 2](https://subscription.packtpub.com/book/programming/9781806380138/2), construiremos sobre estos cimientos sentando primero las bases de los entornos de ejecución de Kubernetes para luego desarrollar la automatización que los hace confiables a escala. Aprenderás a diseñar e implementar canalizaciones de CI/CD para tu plataforma, aplicar flujos de trabajo GitOps para conciliar continuamente el estado del entorno de ejecución e integrar seguridad, políticas y pruebas en cada etapa del despliegue. Aquí es donde la plataforma comienza a sentirse menos como una colección de clústeres de Kubernetes y más como un producto independiente que los equipos pueden utilizar de forma segura y predecible para crear su software de dominio.

---

### Enlaces de Referencia

- **[1]** *Effective Platform Engineering* - Manning Publications [2025], Chankramath A, Alvarez P, et al., [effectiveplatformengineering.com](https://effectiveplatformengineering.com/)
- **[2]** *Trunk Based Development* [2020] - Hammant, Paul, [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/)
- **[3]** *Pulumi - Infrastructure as Code on any platform* - [pulumi.com](https://pulumi.com/)
- **[4]** *Open Policy Agent* - [openpolicyagent.org](https://openpolicyagent.org/)
- **[5]** *Bitwarden* - [bitwarden.com](https://bitwarden.com/)
- **[6]** *CircleCI* - [circleci.com](https://circleci.com/)
- **[7]** *Kind* - [kind.sigs.k8s.io](https://kind.sigs.k8s.io/)
