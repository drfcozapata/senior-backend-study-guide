# Módulo 00 — Glosario

Glosario vivo de la guía. Crece con cada módulo: cada vez que aparezca un concepto nuevo en cualquier módulo, su definición de referencia se añade aquí. Los módulos enlazan a estas definiciones en lugar de repetirlas.

**Cómo leer este glosario:** cada entrada tiene una definición corta (una frase), una definición práctica (qué implica en sistemas reales) y una referencia al módulo donde se explica en profundidad.

---

## Índice alfabético

[A](#a) · [B](#b) · [C](#c) · [D](#d) · [E](#e) · [F](#f) · [G](#g) · [H](#h) · [I](#i) · [J](#j) · [L](#l) · [M](#m) · [N](#n) · [O](#o) · [P](#p) · [Q](#q) · [R](#r) · [S](#s) · [T](#t) · [V](#v) · [W](#w)

---

## A

### AGENTS.md
**Definición corta:** Archivo de instrucciones persistentes en la raíz del repo que todo agente de IA lee al iniciar: estructura, convenciones, comandos, restricciones.

**En la práctica:** es el *system prompt* por repositorio. El estándar AGENTS.md (OpenAI, ago. 2025; adoptado por Codex, Cursor, Copilot, Gemini CLI, OpenCode…) y CLAUDE.md (Anthropic) cumplen el mismo papel. Bien escrito reduce errores del agente (que no conoce las convenciones), mal escrito o ausente provoca código que no respeta la arquitectura del repo. Se actualiza en el PR que cambia las convenciones (documentación viva).

**Relacionado con:** [Context Engineering](#context-engineering) · **Profundizado en:** Módulo 13

### Agent loop
**Definición corta:** Ciclo *plan → act → observe* que ejecuta un agente de IA hasta completar la tarea: decide una acción (leer archivo, correr comando), la ejecuta, observa el resultado y decide la siguiente.

**En la práctica:** es la unidad arquitectónica de todos los agentes de código (Claude Code, Codex, OpenCode). Dos implicaciones: el **contexto** es el input dominante (solo sabe lo que le pones), y el **compound mistakes** degrada la fiabilidad con cada paso (95% por paso → 60% en 10 pasos). Por eso: specs claras, descomposición y validar el plan antes de ejecutar.

**Relacionado con:** [MCP (Model Context Protocol)](#mcp-model-context-protocol) · [Subagente](#subagente) · **Profundizado en:** Módulo 13

### ACID
**Definición corta:** Conjunto de propiedades (Atomicidad, Consistencia, Aislamiento, Durabilidad) que garantiza que las transacciones de una base de datos se procesen de forma fiable.

**En la práctica:**
- **Atomicidad:** una transacción se ejecuta completamente o no se ejecuta. No hay estados intermedios.
- **Consistencia:** cada transacción lleva la base de datos de un estado válido a otro estado válido (se cumplen todas las restricciones).
- **Aislamiento:** transacciones concurrentes no se ven entre sí; el resultado es equivalente a ejecutarlas en serie.
- **Durabilidad:** una vez confirmada (commit), la transacción sobrevive a fallos del sistema.

Las bases de datos relacionales tradicionales (PostgreSQL, MySQL) ofrecen ACID completo. En sistemas distribuidos, mantener ACID a través de múltiples nodos es costoso o imposible (ver [CAP Theorem](#cap-theorem)), lo que motiva modelos alternativos como [BASE](#base).

**Contraste con:** [BASE](#base) · **Profundizado en:** Módulo 05 (Bases de datos distribuidas)

### Adaptive Bitrate Streaming (ABR)
**Definición corta:** Técnica de streaming que trocea el video en **chunks de ~2 segundos** codificados a **múltiples bitrates** (ladder: 240p–4K); el cliente elige en cada momento el bitrate que su red y buffer soportan (HLS/DASH).

**En la práctica:** un título se codifica en 5-15 bitrates × codecs → decenas de variantes; el cliente pide primero un **manifest** (lista de chunks por calidad) y luego los chunks uno a uno. Es lo que permite ver 4K en fibra y SD en 3G con la misma infraestructura. Consecuencia para diseño: cada stream genera **miles de objetos cacheados** (un 4K ≈ 15-20 GB, 50+ encodings) y los picos de premieres crean **cache miss storms** en el CDN. Ver [CDN](#cdn-content-delivery-network).

**Relacionado con:** [CDN](#cdn-content-delivery-network) · **Profundizado en:** Módulo 14

### At-least-once (semántica de entrega)
**Definición corta:** Garantía de que cada mensaje se entrega **una o más veces**; nunca se pierde, pero puede duplicarse.

**En la práctica:** es la semántica por defecto en la mayoría de sistemas de mensajería (SQS estándar, Kafka con configuración habitual). El consumidor **debe ser idempotente** porque recibirá duplicados. Es el trade-off habitual: prefieres procesar dos veces a perder un evento de pago.

**Relacionado con:** [Idempotencia](#idempotencia) · [At-most-once](#at-most-once-semántica-de-entrega) · [Exactly-once](#exactly-once-semántica-de-entrega) · **Profundizado en:** Módulo 03

### At-most-once (semántica de entrega)
**Definición corta:** Garantía de que cada mensaje se entrega **como máximo una vez**; puede perderse, pero nunca se duplica.

**En la práctica:** adecuada cuando perder un mensaje es aceptable (métricas, logs, telemetría) y el coste de duplicar sería peor. Se logra sin reintentos ni ACKs.

**Profundizado en:** Módulo 03

---

## B

### Back-of-the-envelope estimation
**Definición corta:** Estimación aproximada de escala hecha "en el reverso de un sobre": QPS, almacenamiento, ancho de banda y cache, con potencias de diez y supuestos declarados.

**En la práctica:** es el paso que separa a un candidato senior en entrevistas de diseño. Anclajes que debes memorizar: **~10⁵ segundos por día** (86.400), por lo que *X requests/día ≈ X/10⁵ por segundo*; **factor de pico 2-10×** sobre el promedio; y referencias (500M posts/día en X, 100.000M msg/día en WhatsApp, ~1.25M GPS updates/s en Uber, ~20M streams pico en Netflix). Regla: un orden de magnitud de error es aceptable; lo evaluado es el **razonamiento cuantitativo**, no la cifra exacta. Nunca omitas declarar los supuestos ("asumo 1 KB por post, ratio read:write 100:1").

**Profundizado en:** Módulo 14

### Blue-Green (estrategia de despliegue)
**Definición corta:** Dos entornos de producción idénticos (blue y green); solo uno recibe tráfico en cada momento. El release es cambiar el router/load balancer hacia el otro entorno.

**En la práctica:** despliegas la versión nueva en el entorno inactivo, la pruebas sin interrumpir el tráfico, y luego rediriges el tráfico. **Rollback = volver a dirigir el tráfico al anterior** (instantáneo, sin redeploy). Costo: el doble de infraestructura. Dificultad: la base de datos compartida — se resuelve con cambios de esquema aditivos (la app no asume versión de BD) o dos BDs (con riesgo de perder transacciones al rollback).

**Relacionado con:** [Canary](#canary-estrategia-de-despliegue) · [Feature Flag](#feature-flag-toggle) · **Profundizado en:** Módulo 12

### Backpressure
**Definición corta:** Mecanismo por el cual un sistema señala a sus productores que reduzcan el ritmo cuando no puede procesar la carga entrante.

**En la práctica:** sin backpressure, un consumidor lento provoca colas infinitas, memoria agotada y caídas en cascada. Las estrategias incluyen: buffers acotados, rechazo explícito (load shedding), pausas en el productor, o colas intermedias (SQS) que absorben picos.

**Profundizado en:** Módulo 10

### BASE
**Definición corta:** Modelo de consistencia alternativo a ACID: **B**asically **A**vailable, **S**oft state, **E**ventually consistent.

**En la práctica:** describe el comportamiento de sistemas distribuidos que priorizan disponibilidad sobre consistencia inmediata (ver [CAP Theorem](#cap-theorem)): el sistema está básicamente disponible, su estado puede cambiar incluso sin entrada, y converge a la consistencia con el tiempo. Es el modelo de DynamoDB, Cassandra y la mayoría de NoSQL.

**Contraste con:** [ACID](#acid) · **Relacionado con:** [Consistencia eventual](#consistencia-eventual) · **Profundizado en:** Módulos 05 y 10

---

## C

### Context Engineering
**Definición corta:** Disciplina de gestionar *todo lo que el agente de IA ve* (instrucciones persistentes, estructura del repo, archivos leídos, resultados de ejecución) como el input dominante de la calidad del output.

**En la práctica:** el prompt es una instrucción; el contexto es lo que el agente percibe. Tres patrones: **AGENTS.md/CLAUDE.md** en la raíz (system prompt por repo), **specs en archivos > conversaciones largas** (las sesiones largas degradan: el agente "olvida" el principio), y **contexto justo y a demanda** (repo map + lectura selectiva + subagentes que aíslan contextos). Aunque la ventana llegue a 1M tokens, más contexto no es gratis: costo, latencia y peor retención del "middle of context".

**Relacionado con:** [AGENTS.md](#agentsmd) · [Prompt engineering](#prompt-engineering) · **Profundizado en:** Módulo 13

### Canary (estrategia de despliegue)
**Definición corta:** Promover una release por entornos cada vez más grandes a medida que se confirma que funciona (empleados → pequeño % de clientes → todos), deteniéndose si falla.

**En la práctica:** el nombre viene de los canarios de los mineros: detectan el gas antes que los mineros. Variante avanzada: **cluster immune system**, que conecta el monitoreo de producción con el proceso de release y **automatiza el rollback** si las métricas de usuario se desvían (latencia, error rate, conversión).

**Relacionado con:** [Blue-Green](#blue-green-estrategia-de-despliegue) · **Profundizado en:** Módulo 12

### CI / Continuous Delivery / Continuous Deployment
**Definición corta:** CI integra el código de todos en trunk diariamente con tests automatizados; Continuous Delivery mantiene trunk desplegable y permite liberar a demanda; Continuous Deployment despliega automáticamente cada build bueno.

**En la práctica:** CI es el prerrequisito de CD, y CD lo es de CDel. El matiz clave: **deployment ≠ release** — desplegar es instalar una versión; liberar es hacer una feature visible a usuarios o segmentos. Desacoplarlos permite desplegar frecuente y de bajo riesgo mientras el negocio decide cuándo liberar. CDel aplica sobre todo a servicios web; CD es el estándar en casi todo (mobile, COTS, embedded).

**Profundizado en:** Módulo 12

### Cache-Aside (patrón de caché)
**Definición corta:** Patrón donde la aplicación consulta primero la caché; en caso de miss, lee de la base de datos, guarda en caché y devuelve el dato.

**En la práctica:** es el patrón de caché más común (típicamente con Redis o ElastiCache). Ventajas: simple, la caché solo contiene lo que se usa. Problemas: el primer acceso es lento (cache miss), datos potencialmente obsoletos (requiere TTL o invalidación), y riesgo de *cache stampede* si muchos requests golpean un miss simultáneo (mitigar con locks o request coalescing).

**Relacionado con:** [Read-Through](#read-through-patrón-de-caché) · [Write-Through](#write-through-patrón-de-caché) · [Write-Behind](#write-behind-patrón-de-caché) · **Profundizado en:** Módulo 10

### CAP Theorem
**Definición corta:** En un sistema distribuido, ante una partición de red (**P**), debes elegir entre consistencia (**C**) y disponibilidad (**A**); no puedes tener ambas.

**En la práctica:**
- **CP (consistencia + tolerancia a particiones):** el sistema rechaza operaciones si no puede garantizar datos correctos (ej: HBase, etcd, Zookeeper). Útil para configuración de cluster, locks distribuidos.
- **AP (disponibilidad + tolerancia a particiones):** el sistema responde siempre, aunque los datos puedan estar desactualizados (ej: DynamoDB, Cassandra, CouchDB). Útil para carritos de compra, sesiones, catálogos.

El teorema se malinterpreta a menudo: **C y A solo entran en conflicto durante una partición**. En operación normal puedes tener ambas. Esa matización la formaliza [PACELC](#pacelc).

**Relacionado con:** [PACELC](#pacelc) · [BASE](#base) · [Consistencia eventual](#consistencia-eventual) · **Profundizado en:** Módulos 01, 05 y 10

### CDN (Content Delivery Network)
**Definición corta:** Red distribuida de servidores edge que cachea contenido (estáticos, media, video) cerca del usuario para reducir latencia y carga del origen.

**En la práctica:** el usuario recibe los bytes desde un edge a <10 ms; solo los **cache misses** y el long-tail llegan al origen (en Netflix, ~95-98% del tráfico lo sirve su CDN propia, Open Connect, instalada *dentro de los ISPs*). Decisiones senior: distribución de popularidad **Zipf** (pocos títulos concentran la demanda → dimensiona los tiers), **pre-warming** predictivo para estrenos, retry con backoff+jitter contra la **cache miss storm** (thundering herd de millones de peticiones al origen), y la **elección de no construir tu propio CDN** salvo que el costo por byte lo justifique (Netflix sí, una tienda no).

**Relacionado con:** [Adaptive Bitrate Streaming (ABR)](#adaptive-bitrate-streaming-abr) · **Profundizado en:** Módulo 14

### Circuit Breaker
**Definición corta:** Patrón que envuelve llamadas a un servicio externo: si los fallos superan un umbral, el circuito "se abre" y las llamadas fallan inmediatamente sin intentar la conexión.

**En la práctica:** estados: **Cerrado** (normal, los fallos se cuentan) → **Abierto** (rechazo inmediato, protege al servicio degradado) → **Semi-abierto** (tras un timeout, deja pasar algunas llamadas de prueba; si funcionan, se cierra). Evita que un servicio caído colapse a sus llamadores (agotando sus threads/conexiones) y le da tiempo a recuperarse. Implementaciones: Polly (.NET), Resilience4j (Java), está integrado en service meshes como Istio.

**Relacionado con:** [Retry](#retry) · **Profundizado en:** Módulo 10

### Clean Architecture
**Definición corta:** Arquitectura en capas concéntricas de Robert C. Martin donde las dependencias apuntan siempre hacia el centro (el dominio), nunca hacia afuera.

**En la práctica:** capas de dentro hacia fuera: **Entidades** (reglas de negocio del dominio) → **Casos de uso** (lógica de aplicación) → **Adaptadores de interfaz** (controladores, presentadores, gateways) → **Frameworks y drivers** (HTTP, bases de datos, UI). La regla clave: el dominio no conoce ni importa nada del exterior; puedes cambiar la base de datos o el framework web sin tocar la lógica de negocio.

**Relacionado con:** [Hexagonal Architecture](#hexagonal-architecture) · [Onion Architecture](#onion-architecture) · **Profundizado en:** Módulo 01

### Consistencia eventual
**Definición corta:** Garantía de que, si no hay nuevas escrituras, todas las réplicas de un dato convergerán al mismo valor en algún momento.

**En la práctica:** "eventual" suele ser milisegundos, pero puede ser más. Implica que una lectura inmediatamente después de una escritura puede devolver el valor antiguo (**read-your-writes** no garantizado). Estrategias para manejarlo: lecturas consistentes opcionales (más caras), sticky sessions, versionado de datos, o diseñar la UI para que tolere el delay (ej: "tu pedido se está procesando"). Es el modelo por defecto de DynamoDB (las lecturas fuertemente consistentes cuestan el doble).

**Relacionado con:** [BASE](#base) · [CAP Theorem](#cap-theorem) · **Profundizado en:** Módulos 05 y 10

### Consistent Hashing
**Definición corta:** Técnica de distribución de claves donde añadir o quitar un nodo solo reubica ~1/N de las claves (en lugar de rehashear todo como con `key % N`).

**En la práctica:** nodos y claves se mapean a puntos de un anillo hash (0–2³²); cada clave pertenece al siguiente nodo en el sentido de las agujas del reloj. Los **nodos virtuales** (cada nodo físico ocupa varios puntos del anillo) equilibran la distribución. Lo usan DynamoDB internamente, Cassandra, Memcached, y los balanceadores de caché. Es la respuesta estándar en entrevistas para "¿cómo distribuyes claves en una caché/shard que puede escalar?"

**Relacionado con:** [Sharding](#sharding) · **Profundizado en:** Módulos 05 y 14

### CQRS (Command Query Responsibility Segregation)
**Definición corta:** Patrón que separa el modelo de escritura (commands, que cambian estado) del modelo de lectura (queries, que lo consultan).

**En la práctica:** las escrituras van a un modelo optimizado para consistencia y reglas de negocio; las lecturas van a modelos desnormalizados optimizados para consulta (pueden ser otras tablas, otra base de datos, o vistas materializadas). Los modelos se sincronizan por eventos (perfecto compañero de [Event Sourcing](#event-sourcing)). **Advertencia senior:** CQRS añade mucha complejidad (consistencia eventual entre modelos, más infraestructura); úsalo cuando lecturas y escrituras tienen requisitos radicalmente distintos, no por defecto.

**Relacionado con:** [Event Sourcing](#event-sourcing) · **Profundizado en:** Módulo 03

---

## D

### DORA Metrics
**Definición corta:** Las cuatro métricas clave del informe State of DevOps (DORA/Google Cloud): **Deployment Frequency**, **Lead Time for Changes**, **Change Failure Rate**, **MTTR** (tiempo para restaurar el servicio).

**En la práctica:** distinguen equipos elite/high/medium/low. Despliegues frecuentes y baratos, lead time corto, baja tasa de fallos y recuperación rápida correlacionan con mayor rendimiento organizacional. Las métricas tienen **trade-offs**: desplegar más frecuente con gates mal puestos sube el Change Failure Rate; bajar MTTR a costa de sacrificar la causa raíz es deuda a futuro.

**Relacionado con:** [CI / Continuous Delivery / Continuous Deployment](#ci--continuous-delivery--continuous-deployment) · **Profundizado en:** Módulo 12

### Double-entry ledger (contabilidad de partida doble)
**Definición corta:** Registro financiero **append-only e inmutable** donde cada transacción escribe **dos asientos que suman cero** (un débito y un crédito); los balances se derivan sumando asientos, nunca se almacenan como campo mutable.

**En la práctica:** es el patrón de correctitud de todo sistema de pagos serio (Stripe, bancos). Un cargo de $100 crea débito `customer_receivable +100` y crédito `revenue +100`; un refund crea los asientos inversos. Implicaciones de diseño: si débitos ≠ créditos, algo está mal — y lo sabes al instante; las correcciones se hacen con **asientos de reversión**, nunca editando (audit trail completo); el ledger **casi no se particiona** — aceptas un solo Postgres con `SERIALIZABLE` como bottleneck porque la correctitud pesa más que la escala. La conciliación diaria contra el *settlement file* del procesador prueba que tu vista de la realidad coincide con la del banco.

**Relacionado con:** [Idempotencia](#idempotencia) · [Outbox](#outbox-patrón--transactional-outbox) · **Profundizado en:** Módulo 14

### Deployment vs. Release
**Definición corta:** **Deployment**: instalar una versión de software en un entorno. **Release**: hacer que una funcionalidad sea visible/usada por los usuarios.

**En la práctica:** desacoplar ambos es la clave del despliegue de bajo riesgo: puedes desplegar diario (cambios pequeños, reversibles) y liberar features cuando el negocio quiera, activando un [Feature Flag](#feature-flag-toggle) o una variable de configuración. Con despliegue y release acoplados, cada release es un evento de alto riesgo que necesita ventana de mantenimiento y personal de guardia.

**Relacionado con:** [CI / Continuous Delivery / Continuous Deployment](#ci--continuous-delivery--continuous-deployment) · **Profundizado en:** Módulo 12

### DDD (Domain-Driven Design)
**Definición corta:** Enfoque de diseño de software (Eric Evans) que pone el dominio del negocio en el centro, modelándolo con un lenguaje ubicuo compartido entre técnicos y expertos del negocio.

**En la práctica:**
- **Estratégico:** Bounded Contexts (límites donde un modelo es válido), Context Maps (relaciones entre contextos), Ubiquitous Language.
- **Táctico:** Entities (identidad), Value Objects (inmutables, sin identidad), Aggregates (fronteras de consistencia transaccional), Repositories, Domain Services, Domain Events.

Mapeo clave con microservicios: **cada Bounded Context es un candidato natural para un microservicio**.

**Profundizado en:** Módulo 02

### Distributed Tracing
**Definición corta:** Técnica que sigue una petición a través de todos los servicios que toca, asignándole un trace ID único que se propaga en los headers.

**En la práctica:** un *trace* se compone de *spans* (unidades de trabajo con inicio, duración y metadata; anidados). El estándar de facto es **OpenTelemetry** (OTel), que unifica traces, métricas y logs; los datos se visualizan en Jaeger, Zipkin, X-Ray, Datadog, etc. Los headers de propagación estándar son W3C Trace Context (`traceparent`). Sin tracing distribuido, depurar latencia en microservicios es adivinanza.

**Relacionado con:** Observabilidad · **Profundizado en:** Módulo 07

---

## E

### Event Sourcing
**Definición corta:** Patrón donde el estado de la aplicación se deriva de una secuencia inmutable de eventos almacenados, en lugar de almacenar solo el estado actual.

**En la práctica:** en lugar de `UPDATE balance = 500`, guardas `DepositoRealizado(200)`, `RetiroRealizado(50)`… El estado actual se reconstruye reproduciendo los eventos (los *snapshots* aceleran el replay). Beneficios: auditoría completa gratis, time-travel (estado en cualquier punto), replay para corregir bugs o repoblar proyecciones. Costes: curva de aprendizaje, migración de eventos (versionado de schema), troubleshooting más difícil. Se combina naturalmente con [CQRS](#cqrs-command-query-responsibility-segregation).

**Profundizado en:** Módulo 03

### Exactly-once (semántica de entrega)
**Definición corta:** Garantía (aparente) de que cada mensaje se procesa exactamente una vez: ni perdido ni duplicado.

**En la práctica — lo que un senior debe saber:** **exactly-once real de extremo a extremo no existe** en sistemas distribuidos generales (es una consecuencia del problema de los dos generales). Lo que sí existe:
- **Effectively-once:** combinación de deduplicación (IDs de mensaje) + [idempotencia](#idempotencia) en el consumidor, que *se comporta como* exactly-once.
- **Exactly-once acotado:** dentro de un sistema concreto (Kafka con transacciones entre topics, SQS FIFO con Message Deduplication ID).

La respuesta correcta en producción es casi siempre: at-least-once + idempotencia.

**Relacionado con:** [At-least-once](#at-least-once-semántica-de-entrega) · [At-most-once](#at-most-once-semántica-de-entrega) · [Idempotencia](#idempotencia) · **Profundizado en:** Módulo 03

---

## F

### (Fallacies of Distributed Computing)
**Definición corta:** Ocho supuestos falsos que los programadores tienden a hacer sobre sistemas distribuidos.

**Las ocho:** la red es confiable; la latencia es cero; el ancho de banda es infinito; la red es segura; la topología no cambia; hay un solo administrador; el coste de transporte es cero; la red es homogénea. Diseñar como si fueran ciertas produce sistemas frágiles: timeouts ausentes, reintentos sin límite, chatty APIs, datos sensibles sin cifrar.

**Profundizado en:** Módulo 01

### Fan-out
**Definición corta:** Factor por el cual una operación inicial se multiplica en varias operaciones aguas abajo (1 post → N timelines, 1 evento → N notificaciones). Es la decisión central de los sistemas de feed.

**En la práctica:** dos estrategias. **Pull:** el reader consulta al leer (simple, pero inviable con ratio read:write alto — en X sería 400M lookups/s). **Push:** al escribir, se entrega a todos los interesados (writes caros: 5.800 posts/s × 200 followers ≈ 1M writes/s, pero reads triviales). Con ratio 100:1 pagas en el write. La variante **híbrida** resuelve el hot key: las cuentas con millones de followers no se fanoutean, se almacenan aparte y se **mergean en lectura**. El [Two-stage fanout](#two-stage-fanout) extiende el patrón a campañas masivas.

**Relacionado con:** [Materialized view](#materialized-view) · [Two-stage fanout](#two-stage-fanout) · **Profundizado en:** Módulo 14

### Feature Flag (toggle)
**Definición corta:** Mecanismo que enciende/apaga una funcionalidad en tiempo de ejecución (configuración, no deploy), sin desplegar código nuevo.

**En la práctica:** desacopla **deployment de release** (ver [Deployment vs. Release](#deployment-vs-release)): despliegas el código con la feature apagada y la activas cuando quieras, por usuario, porcentaje o segmento. Permite release progresivo, canary de negocio, kill switch instantáneo y rollback sin redeploy. Requisitos: **features muertas se eliminan** (deuda técnica), flags de corta vida para releases y de larga vida para operaciones, y gestionarlas bien (LaunchDarkly, o en casa con configuración + feature flag service) para no convertir la app en un árbol de ifs.

**Relacionado con:** [Canary](#canary-estrategia-de-despliegue) · [Deployment vs. Release](#deployment-vs-release) · **Profundizado en:** Módulo 12

---

## G

### Geospatial indexing (S2 / Geohash / H3)
**Definición corta:** Técnica de indexación geoespacial que divide el mapa en **celdas** con un ID único; la celda es a la vez la clave de shard y el índice para consultas de proximidad (k-nearest-neighbors).

**En la práctica:** responde "¿qué drivers están cerca?" a millones de actualizaciones de GPS por segundo, algo que una base de datos relacional no hace. La celda del rider + las vecinas dan los candidatos en O(1) por partición. Diferencias: **Geohash** (simple, celdas de forma irregular), **Google S2** (curvas de Hilbert, cobertura uniforme), **Uber H3** (hexágonos de vecindad uniforme). Decisiones senior: el estado actual (última ubicación) vive en un **índice in-memory** (overwrite, el dato viejo no importa) mientras el histórico va al camino frío; y el **hot cell** (Times Square) se mitiga con replicación/sub-sharding de la celda caliente.

**Relacionado con:** [Fan-out](#fan-out) · **Profundizado en:** Módulo 14

### GitOps
**Definición corta:** Patrón donde Git es la **única fuente de verdad** del estado deseado de la infraestructura y las aplicaciones; un operador en el cluster reconcilia continuamente el estado real con el declarado en el repo.

**En la práctica:** los cuatro principios: estado deseado declarativo, versionado en Git, cambios por PR con revisión y CI, y reconciliación automática (pull). Push vs. pull: en **pull**, el agente dentro del cluster observa el repo (ArgoCD/Flux) y no requiere exponer credenciales de cluster a CI/CD; el agente detecta **drift** (config cambia por fuera de Git) y lo corrige. ArgoCD (App of Apps, aplicaciones progresivas) vs. Flux (Kustomize/Kubernetes controllers). Secretos: Sealed Secrets, SOPS, External Secrets Operator.

**Relacionado con:** [Rollback](#rollback--fix-forward) · **Profundizado en:** Módulo 12

---

## H

### Hexagonal Architecture (Ports & Adapters)
**Definición corta:** Arquitectura (Alistair Cockburn) donde el núcleo de la aplicación se aísla del mundo exterior mediante **puertos** (interfaces) y **adaptadores** (implementaciones concretas).

**En la práctica:** el dominio define puertos de entrada (driving: lo que la app ofrece, ej: `OrderService`) y de salida (driven: lo que la app necesita, ej: `OrderRepository`, `PaymentGateway`). Los adaptadores implementan los puertos para tecnologías concretas: REST controller, PostgreSQL, Stripe, SQS. Puedes intercambiar adaptadores (y probar el dominio con adaptadores de test) sin tocar el núcleo. Conceptualmente equivalente a [Clean Architecture](#clean-architecture) y [Onion Architecture](#onion-architecture): las tres son "arquitecturas centradas en el dominio".

**Profundizado en:** Módulo 01

### Hot key / hot partition / hot cell
**Definición corta:** Una clave (o shard, o celda geográfica) que concentra una fracción desproporcionada del tráfico, sobrecargando el nodo que la sirve.

**En la práctica:** es el modo de fallo de escala más común y el que un senior debe anticipar. Ejemplos: una celebridad con 100M de followers (fan-out), una celda de Uber en Times Square, una partición de base de datos con el producto estrella. Mitigaciones: **sub-sharding** o replicación extra de la clave caliente, **merge en lectura** en lugar de fan-out para celebridades, distribuir el hashing de esa clave, y cachear agresivamente el dato caliente. En una entrevista de diseño, *nombrar el hot key y su mitigación sin que te lo pregunten* es señal de senior.

**Relacionado con:** [Fan-out](#fan-out) · [Sharding](#sharding) · **Profundizado en:** Módulo 14

---

## I

### Idempotencia
**Definición corta:** Propiedad de una operación por la cual ejecutarla múltiples veces produce el mismo resultado que ejecutarla una vez.

**En la práctica:** es *la* propiedad más importante en sistemas distribuidos, porque la red reintenta, los mensajes se duplican ([at-least-once](#at-least-once-semántica-de-entrega)) y los usuarios hacen doble clic. Técnicas:
- **Idempotency keys:** el cliente genera un ID único por operación (Stripe lo popularizó en su API); el servidor guarda el resultado y devuelve la misma respuesta ante repeticiones.
- **Deduplicación:** SQS FIFO, Kafka con IDs.
- **Operaciones naturalmente idempotentes:** `PUT`, `DELETE` en REST; upserts en bases de datos.
- **Claves de condición:** conditional writes en DynamoDB.

**Relacionado con:** [At-least-once](#at-least-once-semántica-de-entrega) · [Exactly-once](#exactly-once-semántica-de-entrega) · **Profundizado en:** Módulos 03 y 09

---

## J

### JWT (JSON Web Token)
**Definición corta:** Estándar (RFC 7519) para transmitir claims firmados como JSON compacto, con estructura `header.payload.signature`.

**En la práctica:** el servidor emite un JWT firmado (típicamente RS256/ES256 con clave asimétrica) tras autenticar al usuario; el cliente lo envía en `Authorization: Bearer <token>`; los servicios lo validan localmente con la clave pública, sin llamar al servidor de auth. Claims estándar: `iss`, `sub`, `aud`, `exp`, `iat`. Puntos críticos de seguridad: validar siempre firma y `exp`/`aud`, nunca confiar en `alg: none`, tokens de corta vida + refresh tokens, y recordar que **no se pueden revocar individualmente** sin estado adicional (blocklist o versiones).

**Relacionado con:** OAuth2 / OIDC · **Profundizado en:** Módulo 08

---

## L

### Leader Election
**Definición corta:** Proceso por el cual los nodos de un cluster acuerdan cuál de ellos ejerce como coordinador (leader).

**En la práctica:** el leader se usa para decisiones que requieren un único punto de decisión: asignar trabajos, secuenciar escrituras, coordinación. Algoritmos: Raft, Paxos, o servicios de consenso externos (etcd, Zookeeper) con leases. Trade-offs: el leader puede ser cuello de botella y single point of failure (mitigado con re-elección automática). En entrevistas, es la respuesta a "¿cómo coordinas N workers para que solo uno ejecute cada tarea?"

**Relacionado con:** [Leader/Follower](#leaderfollower-replication) · **Profundizado en:** Módulo 10

### Leader/Follower (replication)
**Definición corta:** Modelo de replicación donde un nodo (leader) acepta todas las escrituras y las propaga a los followers, que sirven lecturas.

**En la práctica:** la replicación puede ser **síncrona** (el leader confirma solo cuando el follower persiste: más consistente, más latente) o **asíncrona** (el leader confirma inmediatamente: más rápido, riesgo de perder datos si el leader cae). Problemas clásicos: replication lag (el follower devuelve datos viejos → [consistencia eventual](#consistencia-eventual)), y lo que pasa si el leader muere con escrituras no replicadas (split-brain → [Leader Election](#leader-election)). Así funcionan PostgreSQL (streaming replication), MySQL, y las read replicas de RDS.

**Relacionado con:** [Replication](#replication) · **Profundizado en:** Módulos 05 y 10

---

## M

### Materialized view
**Definición corta:** Resultado de una consulta **precomputado y almacenado** como dato derivado (feed, timeline cache, balances agregados), en lugar de calcularse en cada lectura.

**En la práctica:** el ejemplo canónico es el home timeline de X: en vez de consultar los posts de 200 follows en cada lectura (inviable), el post se inserta en el timeline materializado (mailbox) de cada follower al publicarse, y el feed se sirve desde cache. **Trade-off fundamental:** velociza los reads a cambio de más trabajo en los writes (write-amplification). Se elige cuando el cálculo es caro y el ratio read/write es alto. El patrón [Fan-out](#fan-out) es la mecánica de actualización de la vista materializada. Los balances financieros son el caso "derivado" equivalente (ver [Double-entry ledger](#double-entry-ledger-contabilidad-de-partida-doble)).

**Relacionado con:** [Fan-out](#fan-out) · [CQRS](#cqrs-command-query-responsibility-segregation) · [Double-entry ledger](#double-entry-ledger-contabilidad-de-partida-doble) · **Profundizado en:** Módulo 14

### MCP (Model Context Protocol)
**Definición corta:** Estándar abierto (Anthropic, 2024; JSON-RPC 2.0) que estandariza cómo los LLM descubren e invocan herramientas externas. Resuelve el problema *N×M* de integraciones convirtiéndolo en *N+M*.

**En la práctica:** arquitectura **host/client/server** (el host es la app con el LLM — Claude Code, Cursor, OpenCode, VS Code; el server expone capacidades). Primitivas: **tools** (acciones ejecutables), **resources** (datos de solo lectura), **prompts** (plantillas). Transports: `stdio` (local) y `HTTP/SSE` (remoto). El matiz senior: MCP le da a los agentes *write actions* (pueden ejecutar código), y la especificación advierte que las herramientas son "arbitrary code execution" y sus descripciones se tratan como **no confiables** — es el vector del **agentjacking** (instrucciones inyectadas en datos que el agente ejecuta con los privilegios del usuario).

**Relacionado con:** [Agent loop](#agent-loop) · [Prompt injection](#prompt-injection) · **Profundizado en:** Módulo 13

### (Microservicios)
**Definición corta:** Estilo arquitectónico donde la aplicación se estructura como servicios pequeños, autónomos, desplegables independientemente, modelados alrededor de dominios de negocio.

**En la práctica:** las ventajas (despliegue y escalado independientes, equipos autónomos, heterogeneidad tecnológica) tienen un precio: complejidad operativa, red entre componentes, transacciones distribuidas ([Saga](#saga)), observabilidad obligatoria. Regla senior: *no empieces con microservicios*; un monolito modular bien hecho es mejor punto de partida, y los límites correctos vienen de [DDD](#ddd-domain-driven-design).

**Profundizado en:** Módulo 02

---

## N

### (NoSQL)
**Definición corta:** Familia de bases de datos que abandonan el modelo relacional para ganar escalabilidad horizontal, flexibilidad de esquema o rendimiento en cargas específicas.

**Tipos:** clave-valor (DynamoDB, Redis), documentales (MongoDB, DocumentDB), columnares (Cassandra, HBase), grafos (Neo4j, Neptune). El punto clave: NoSQL no significa "sin estructura", significa **modelar según los patrones de acceso** en lugar de normalizar; y aceptar trade-offs de consistencia ([BASE](#base)) a cambio de escala y disponibilidad ([CAP](#cap-theorem)).

**Profundizado en:** Módulo 05

---

## O

### Observabilidad
**Definición corta:** Capacidad de entender el estado interno de un sistema a partir de sus salidas externas: métricas, logs y traces (los "tres pilares").

**En la práctica:** monitoring te dice *que* algo falla (dashboards sobre fallos conocidos); observabilidad te deja investigar *por qué* falla algo que nunca habías visto. En sistemas distribuidos requiere: [distributed tracing](#distributed-tracing), logs estructurados con correlation IDs, métricas RED (Rate, Errors, Duration) por servicio, y SLIs/SLOs en lugar de alarmas arbitrarias.

**Profundizado en:** Módulo 07

### Onion Architecture
**Definición corta:** Arquitectura en capas concéntricas (Jeffrey Palermo) donde el núcleo es el modelo de dominio, rodeado por servicios de dominio, servicios de aplicación y finalmente infraestructura/UI.

**En la práctica:** conceptualmente equivalente a [Hexagonal](#hexagonal-architecture-ports--adapters) y [Clean Architecture](#clean-architecture): las dependencias apuntan hacia adentro; la infraestructura depende del dominio, nunca al revés; el orden de las capas interiores difiere ligeramente entre las tres variantes, pero el principio es el mismo.

**Profundizado en:** Módulo 01

### Outbox (patrón / Transactional Outbox)
**Definición corta:** Patrón para publicar eventos de forma fiable: el evento se guarda en una tabla "outbox" **en la misma transacción** que el cambio de estado, y un proceso separado lo publica al bus de mensajes.

**En la práctica:** resuelve el problema del **dual write**: si escribes en la base de datos y luego publicas a SQS/Kafka (o al revés), un fallo entre ambos pasos deja el sistema inconsistente (el pago se guardó pero nunca se emitió el evento, o viceversa). Con outbox, la escritura es atómica; un relay (CDC con Debezium, o un Lambda que hace polling) publica después, garantizando [at-least-once](#at-least-once-semántica-de-entrega). Los consumidores deben ser [idempotentes](#idempotencia).

**Relacionado con:** [Saga](#saga) · **Profundizado en:** Módulo 03

### OIDC (OpenID Connect) para CI/CD
**Definición corta:** Protocolo que permite a los runners de CI/CD (GitHub Actions, GitLab CI) obtener credenciales **temporales y sin secretos** para acceder a los recursos de un proveedor cloud (AWS, GCP, Azure).

**En la práctica:** el cloud publica un **OIDC provider**; el runner obtiene un token JWT firmado y lo canjea por credenciales de corta duración (STS en AWS). Elimina el **secret estático de largo plazo** guardado en el repo/CI (que era el vector de robo clásico). Se combina con **roles con least-privilege y conditions**: solo el repo/branch/environment correcto puede asumir el rol (ej: `repo: org/repo:ref:refs/heads/main`). Es la práctica recomendada frente a access keys clásicas.

**Relacionado con:** [GitOps](#gitops) · [CI / Continuous Delivery / Continuous Deployment](#ci--continuous-delivery--continuous-deployment) · **Profundizado en:** Módulo 12

---

## P

### Prompt engineering
**Definición corta:** Diseño de las instrucciones que se dan a un modelo de lenguaje (qué hacer, con qué ejemplos y qué formato de salida) para obtener el comportamiento deseado.

**En la práctica:** las técnicas que sobreviven a los modelos: instrucciones claras y sin ambigüedad, ejemplos (*few-shot*), formato de salida explícito, descomponer tareas complejas, y dar tiempo al modelo a "pensar" (chain-of-thought). En código: el prompt define *el cambio*, no solo la tarea ("refactoriza a patrón Strategy; mantén la API; no cambies comportamiento"). Los prompts se versionan y se separan del código (reutilizables, testeables). Diferencia clave: el **prompt** es la instrucción; el **contexto** (ver [Context Engineering](#context-engineering)) es todo lo que el agente percibe.

**Relacionado con:** [Context Engineering](#context-engineering) · [Prompt injection](#prompt-injection) · **Profundizado en:** Módulo 13

### Prompt injection
**Definición corta:** Ataque que inyecta instrucciones maliciosas en los datos que un modelo/agente procesa, para que ejecute acciones no autorizadas (jailbreaking aplicado a entradas indirectas).

**En la práctica:** el modelo no distingue fiablemente entre instrucciones legítimas (system prompt) e instrucciones que vienen de *datos* (web, logs, archivos). En agentes de código el riesgo se amplifica: el **agentjacking** (CSA/Tenet, 2026) inyectó instrucciones en eventos de error de Sentry que el agente leía vía MCP, y el agente **ejecutó comandos del atacante con los privilegios del desarrollador** (85% de éxito en Claude Code/Cursor/Codex CLI), exfiltrando credenciales. Defensas: tratar datos como no autoritativos, MCP servers verificados, permisos mínimos, aprobación humana para *write actions*, sandbox, y no exponer secretos innecesarios.

**Relacionado con:** [MCP (Model Context Protocol)](#mcp-model-context-protocol) · **Profundizado en:** Módulo 13

### PACELC
**Definición corta:** Extensión del teorema CAP: **si hay Partición**, eliges entre Disponibilidad (A) y Consistencia (C); **en caso contrario (Else)**, eliges entre Latencia (L) y Consistencia (C).

**En la práctica:** CAP solo cubre el comportamiento durante particiones; PACELC captura el trade-off del día a día: replicación síncrosa (consistente pero lenta) vs. asíncrona (rápida pero eventualmente consistente). Ejemplos: DynamoDB es PA/EL (disponible y baja latencia por defecto); PostgreSQL con replicación síncrona es PC/EC; muchos sistemas permiten elegir **por operación** (lectura consistente en DynamoDB cuesta el doble de RCUs).

**Relacionado con:** [CAP Theorem](#cap-theorem) · [Consistencia eventual](#consistencia-eventual) · **Profundizado en:** Módulos 05 y 10

---

## Q

### Quality Gate
**Definición corta:** Conjunto de checks automáticos que un build debe pasar antes de poder promover a producción (o a un entorno).

**En la práctica:** la pirámide: lint + unit tests + cobertura → SAST (escaneo estático de seguridad) → SCA/dependency review → container scan + SBOM → DAST → contract tests → smoke tests en producción → análisis de SLO tras el deploy. Cada gate detiene el pipeline con feedback rápido y **solo bloquea lo crítico** (un gate demasiado estricto ralentiza el CI; demasiado laxo no protege). Decide cuáles son blocking vs. informativo.

**Relacionado con:** [CI / Continuous Delivery / Continuous Deployment](#ci--continuous-delivery--continuous-deployment) · **Profundizado en:** Módulo 12

---

## R

### Rate Limiting
**Definición corta:** Control del número de peticiones que un cliente puede hacer en una ventana de tiempo.

**En la práctica:** algoritmos clásicos: **token bucket** (ráfagas permitidas hasta vaciar el bucket — el de API Gateway), **leaky bucket** (ritmo de salida constante), **fixed window** (simple, con problema en los bordes de ventana), **sliding window log/counter** (preciso, más memoria). Respuestas: `429 Too Many Requests` con header `Retry-After`. Decisiones senior: por usuario vs. por IP vs. global, dónde aplicarlo (borde con API Gateway/WAF vs. aplicación), y límites diferenciados por plan (free vs. premium).

**Profundizado en:** Módulos 09 y 10

### Reconciliation (conciliación)
**Definición corta:** Proceso que compara dos fuentes de verdad (el ledger interno y el *settlement file* del procesador/banca) y detecta discrepancias, que saltan a revisión manual.

**En la práctica:** es la tercera garantía no negociable de un sistema de pagos (tras [idempotencia](#idempotencia) y [double-entry ledger](#double-entry-ledger-contabilidad-de-partida-doble)): el sistema puede ser correcto por diseño, pero solo la conciliación **prueba** que tu vista de la realidad coincide con la del banco cada día (batch T+1 a T+3). Señal senior: si un diseño financiero no menciona reconciliación, no está completo. También se aplica como patrón general: comparar estados derivados contra su origen (ej: inventario real vs contado) para detectar drift.

**Relacionado con:** [Double-entry ledger](#double-entry-ledger-contabilidad-de-partida-doble) · [Idempotencia](#idempotencia) · **Profundizado en:** Módulo 14

### Replication
**Definición corta:** Mantener copias del mismo dato en múltiples nodos para mejorar disponibilidad, durabilidad y rendimiento de lectura.

**En la práctica:** modelos principales: [Leader/Follower](#leaderfollower-replication) (un escritor, lectores escalables), multi-leader (escritura en varios nodos; resolución de conflictos), y leaderless (quórum de lectura/escritura: `R + W > N`, estilo Cassandra/Dynamo). Decisiones: síncrona vs. asíncrona (ver [PACELC](#pacelc)), cuántas réplicas, y cómo leer (seguidores con posible lag vs. lecturas consistentes al leader).

**Relacionado con:** [Consistencia eventual](#consistencia-eventual) · [Sharding](#sharding) · **Profundizado en:** Módulos 05 y 10

### Retry
**Definición corta:** Reintentar una operación fallida, típicamente con backoff exponencial y jitter (aleatoriedad) para no sincronizar reintentos de muchos clientes.

**En la práctica — reglas senior:**
- Solo reintentar errores **transitorios** (timeout, 5xx, throttling); nunca errores definitivos (4xx de validación).
- Siempre con límite de intentos y **backoff exponencial + jitter** (el patrón de AWS: `min(rand(0, base × 2^intento), tope)`).
- Reintentar solo operaciones [idempotentes](#idempotencia), o asegurar idempotencia por otro medio.
- Combinar con [Circuit Breaker](#circuit-breaker): reintentos contra un servicio caído solo lo hunden más.

**Profundizado en:** Módulo 10

### Rollback / Fix Forward
**Definición corta:** Estrategias de recuperación tras un release fallido: **rollback** = volver a la versión anterior; **fix forward** = desplegar una corrección que avanza hacia adelante.

**En la práctica:** la elección depende de la estrategia de despliegue y del estado de la base de datos:
- **Blue-Green:** rollback es re-dirigir tráfico al entorno anterior (instantáneo).
- **Canary:** detener la promoción y volcar tráfico a la versión buena.
- **Feature flag:** apagar el flag (sin redeploy).
- **git revert / re-deploy:** desplegar la versión anterior — *solo* si el esquema de BD no cambió incompatiblemente; con migraciones destructivas el rollback no basta y el **fix forward** es la vía (el clásico: "no se puede revertir una migración destructiva").

**Relacionado con:** [Blue-Green](#blue-green-estrategia-de-despliegue) · [Canary](#canary-estrategia-de-despliegue) · [Deployment vs. Release](#deployment-vs-release) · **Profundizado en:** Módulo 12

### Runner (GitHub Actions)
**Definición corta:** Máquina que ejecuta los jobs de un workflow de GitHub Actions. Puede ser hosted (escala automática) o self-hosted (tu propia infraestructura, incluso efímera).

**En la práctica:** los **self-hosted ephemeral** (contenedor/VMs desechables por job) son el estándar de seguridad: cada job corre limpio, sin credenciales persistentes y sin contaminación entre jobs; los permanentes self-hosted son un **riesgo** (un workflow malicioso puede ejecutar código con sus credenciales y acceso a red). Decisión senior: hosted para la mayoría; self-hosted efímero para requerimientos (hardware especial, red privada, costo).

**Relacionado con:** [CI / Continuous Delivery / Continuous Deployment](#ci--continuous-delivery--continuous-deployment) · **Profundizado en:** Módulo 12

---

## S

### Subagente
**Definición corta:** Agente hijo con su propia ventana de contexto y tarea, orquestado por el agente principal de un sistema multi-agente.

**En la práctica:** permite **paralelismo** (varios subagentes trabajan a la vez) y **aislamiento de contexto** (el subagente de tests no llena la ventana del de arquitectura). Patrones: separación *plan/act* (un agente planifica, otro ejecuta — el patrón Architect/Editor), y especialización (tests, migraciones, frontend). Advertencias: los sistemas multi-agente son **más difíciles de depurar** (menos visibilidad), y la orquestación debe validar el plan antes de ejecutar (Huyen: *"planning should be decoupled from execution"*). Úsalo con criterio, no como religión.

**Relacionado con:** [Agent loop](#agent-loop) · **Profundizado en:** Módulo 13

### SWE-bench / Terminal-Bench
**Definición corta:** Benchmarks de agentes de código: SWE-bench mide la resolución de issues reales de GitHub (patch correcto contra la test suite del repo); Terminal-Bench mide tareas reales de terminal en un sandbox.

**En la práctica:** SWE-bench Verified pasó de ~13% (2024) a ~78–80% (2026); Terminal-Bench ~52–58%. Pero la **tasa de PRs aceptados en producción real se estima en 35–50%** — el benchmark no captura convenciones ni expectativas de tu repo. Lectura senior: úsalos para *calibrar expectativas*, nunca como promesa; evalúa tu caso con *evals sobre tu propio código* y métricas de flujo (lead time, change failure rate, costo total de tokens + revisión).

**Relacionado con:** [Context Engineering](#context-engineering) · **Profundizado en:** Módulo 13

### Saga
**Definición corta:** Patrón para mantener consistencia en transacciones que cruzan varios servicios: una secuencia de transacciones locales donde cada paso tiene una **transacción de compensación** que deshace su efecto si falla un paso posterior.

**En la práctica — ejemplo:** pedido de delivery: 1) reservar pago → 2) confirmar restaurante → 3) asignar repartidor. Si falla (3), se ejecutan las compensaciones: cancelar reserva con el restaurante, liberar el pago. Dos estilos:
- **Coreografía:** cada servicio emite eventos y escucha los de otros; descentralizado, simple con pocos pasos, difícil de rastrear con muchos.
- **Orquestación:** un orquestador (Step Functions es ideal) dirige los pasos; más visible y testeable, con un punto lógico central.

No hay rollback real (los datos ya fueron commit): la compensación es una operación de negocio ("reembolsar"), no un `ROLLBACK`.

**Relacionado con:** [Outbox](#outbox-patrón--transactional-outbox) · **Profundizado en:** Módulo 03

### Service Discovery
**Definición corta:** Mecanismo por el que los servicios encuentran las ubicaciones (IP:puerto) de otros servicios dinámicamente, sin configuración hardcodeada.

**En la práctica:** en entornos cloud/containers las IPs cambian constantemente. Modelos: **client-side** (el cliente consulta el registro — Eureka) vs. **server-side** (un balanceador consulta el registro — ALB + Cloud Map en AWS). En AWS: Cloud Map (DNS/API), registros en ECS, o la resolución automática de un service mesh (App Mesh). En Kubernetes, los Services + DNS interno lo resuelven nativamente.

**Profundizado en:** Módulo 02

### Sharding
**Definición corta:** Particionado horizontal de datos: dividir un dataset entre múltiples nodos/shards, cada uno con un subconjunto.

**En la práctica:** la decisión crítica es la **shard key**: debe distribuir uniformemente (evitar hot shards) y estar presente en las consultas frecuentes (las consultas cross-shard son caras). Estrategias: por rango (fácil de recorrer, riesgo de hotspots), por hash (uniforme, no ordenado — ver [Consistent Hashing](#consistent-hashing)), por directorio (flexible, con paso extra). En DynamoDB es automático vía Partition Key; entender sus hot partitions es un tema central del Módulo 05.

**Relacionado con:** [Replication](#replication) · **Profundizado en:** Módulos 05 y 10

### SOLID
**Definición corta:** Cinco principios de diseño orientado a objetos (Robert C. Martin): Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.

**En la práctica:** a nivel senior se espera aplicarlos más allá de las clases: SRP → un microservicio tiene una razón de cambio; OCP → extender vía plugins/adaptadores (ver [Hexagonal](#hexagonal-architecture-ports--adapters)); LSP → las implementaciones de un puerto deben ser intercambiables; ISP → APIs pequeñas y específicas por cliente; DIP → el dominio nunca depende de infraestructura. SOLID es, en el fondo, por qué funcionan las arquitecturas centradas en el dominio.

**Profundizado en:** Módulo 01

---

## T

### Thundering herd (estampida)
**Definición corta:** Cuando una gran cantidad de clientes reintenta o pide simultáneamente el mismo recurso tras un fallo, provocando una avalancha que tumba el sistema (origen, base de datos o cache).

**En la práctica:** dos variantes clásicas. **Retry storm:** millones de clientes con timeout reintentan a la vez → se usa **backoff exponencial + jitter** para descorrelacionar. **Cache stampede:** un cache miss masivo (cold start, eviction, estreno) dispara millones de recalculos del mismo dato hacia el origen → mitigaciones: *single flight* (una sola petición recalcula, el resto espera), pre-warming predictivo (empujar contenido antes del pico), y canary para validar hit ratio. Es un **efecto de segundo orden** que un senior debe nombrar en el deep dive de cualquier diseño con picos predecibles (premieres, Black Friday, elecciones).

**Relacionado con:** [Retry](#retry) · [CDN](#cdn-content-delivery-network) · [Hot key](#hot-key--hot-partition--hot-cell) · **Profundizado en:** Módulos 10 y 14

### Twelve-Factor App
**Definición corta:** Metodología de Heroku con 12 prácticas para construir aplicaciones cloud/SaaS portables y escalables.

**Las 12 (resumen senior):** codebase única en control de versiones; dependencias explícitas; **configuración en variables de entorno** (nunca en el código); backing services como recursos adjuntos; build/release/run separados estrictamente; procesos **stateless** (el estado vive en backing services); port binding (la app se autoexpone); concurrencia por escalado de procesos; **disposability** (arranque rápido, shutdown limpio); paridad dev/staging/prod; logs como flujos de eventos a stdout; procesos admin como one-off. Es la base filosófica de containers, serverless y Kubernetes.

**Profundizado en:** Módulo 01

### Two-stage fanout
**Definición corta:** Patrón para entregar un evento a **decenas de millones** de destinatarios sin bloquear el hot path: primero se encola un job por segmento; unos workers **expanden** cada segmento en mensajes por-usuario en batches.

**En la práctica:** una campaña de 100M de notificaciones no se hace con un loop síncrono ni con 100M mensajes a la vez. El Campaign Service publica **un job por segmento**; el Fanout Service pagina los usuarios de 1.000 en 1.000 y publica por-usuario en la cola del canal, con prioridad para los activos (últimas 24h). El hot path (una notificación transaccional) **nunca** pasa por aquí. Es la extensión de [Fan-out](#fan-out) para el caso de broadcast masivo y el patrón que distingue un diseño de notificaciones mid de uno senior.

**Relacionado con:** [Fan-out](#fan-out) · [Rate Limiting](#rate-limiting) · **Profundizado en:** Módulo 14

---

## V

### Vibe coding
**Definición corta:** (Coloquial) "Codear a vibra": aceptar el output de una IA sin entenderlo, revisarlo ni testearlo, confiando en que "funciona".

**En la práctica:** produce **deuda técnica invisible** (código que nadie puede mantener porque nadie lo entiende), bugs no detectados que pasan a producción y dependencias sin revisar. Es el antónimo del trabajo senior. Lo que lo reemplaza es el *agentic engineering*: dirigir agentes con especificación clara, contexto persistente (AGENTS.md, specs), descomposición y **verificación continua** (tests, CI, review humano del diff). La IA es una herramienta de alta productividad *con* proceso de calidad fuerte; una máquina de deuda sin él.

**Relacionado con:** [Context Engineering](#context-engineering) · **Profundizado en:** Módulo 13

### Vertical Slice Architecture**Definición corta:** Organización del código por **features** (cada slice contiene todo lo necesario para una funcionalidad: endpoint, lógica, acceso a datos) en lugar de por capas técnicas.

**En la práctica:** contraste con capas horizontales (`Controllers/`, `Services/`, `Repositories/`). Cada feature es un módulo autocontenido que acopla mínimamente con otros; añadir una feature = añadir una carpeta, sin tocar código existente. Combina bien con el patrón Mediator (MediatR) y con microservicios, donde cada servicio ya es un slice del negocio. Regla práctica: usa organización por capas dentro de cada slice si la feature es compleja; no fuerces capas globales.

**Relacionado con:** [Clean Architecture](#clean-architecture) · **Profundizado en:** Módulo 01

---

## W

### WebSocket
**Definición corta:** Protocolo que mantiene una **conexión TCP bidireccional y persistente** entre cliente y servidor, permitiendo al servidor *empujar* datos sin polling (RFC 6455).

**En la práctica:** la base del real-time: chat (WhatsApp), presencia, tracking en vivo (Uber), dashboards, juegos. Consecuencias de diseño: las conexiones son **estado** (sticky LB o routing por hash para reencaminar), consumen memoria/CPU por conexión (un thread por conexión no escala — WhatsApp usa procesos livianos de Erlang/BEAM, ~2M conexiones por servidor), y el servidor debe manejar reconnect + reenvíos (dedup por id de mensaje). Alternativa para **unidireccional** server→client: **SSE** (Server-Sent Events, más simple, sin soporte bidireccional). Elección senior: WebSocket cuando hay ida y vuelta; SSE cuando el cliente solo escucha.

**Relacionado con:** [Hot key](#hot-key--hot-partition--hot-cell) · **Profundizado en:** Módulo 14

### Write-Behind / Write-Back (patrón de caché)
**Definición corta:** La aplicación escribe solo en la caché; la persistencia en la base de datos se hace de forma asíncrona después.

**En la práctica:** escrituras muy rápidas y se absorben picos (buffer natural), pero hay **riesgo de pérdida de datos** si la caché cae antes del flush, y la complejidad aumenta (colas, orden, reconciliación). Apropiado para escrituras masivas tolerantes a pérdida (contadores, telemetría, likes).

**Relacionado con:** [Write-Through](#write-through-patrón-de-caché) · [Cache-Aside](#cache-aside-patrón-de-caché) · **Profundizado en:** Módulo 10

### Write-Through (patrón de caché)
**Definición corta:** Cada escritura se aplica a la caché y a la base de datos de forma síncrona (típicamente, la librería de caché lo hace transparente).

**En la práctica:** la caché siempre está fresca y las lecturas nunca dan miss tras una escritura, pero la latencia de escritura aumenta y se cachean datos que quizá nunca se lean (combinable con TTL). Compañero natural de [Read-Through](#read-through-patrón-de-caché).

**Relacionado con:** [Cache-Aside](#cache-aside-patrón-de-caché) · **Profundizado en:** Módulo 10

---

### Read-Through (patrón de caché)
**Definición corta:** En un cache miss, es la propia capa de caché (no la aplicación) la que lee de la base de datos, guarda el resultado y lo devuelve.

**En la práctica:** la aplicación simplifica su lógica (solo habla con la caché) y el *cache stampede* se mitiga porque la capa de caché puede coordinar el miss. Requiere que el proveedor de caché soporte el patrón (o una librería intermedia); menos común que [Cache-Aside](#cache-aside-patrón-de-caché) en stacks cloud, donde Redis/Memcached son almacenes "tontos".

**Profundizado en:** Módulo 10
