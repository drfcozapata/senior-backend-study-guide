# Módulo 15 — Preguntas técnicas de entrevista + ejercicios

> **Nivel:** Senior. Este módulo es el **banco de preguntas** de la guía. No es teoría nueva: es el *cómo se pregunta* todo lo que ya viste en los módulos 01–14, organizado por nivel (Junior / Mid / Senior), y con la particularidad central: **todas las respuestas, incluso las de preguntas junior, son las que daría un ingeniero senior**. Porque la diferencia entre un junior y un senior no es solo el conocimiento — es cómo se *razona* la respuesta: el senior contesta una pregunta básica con la profundidad justa, nombra los trade-offs, ata la decisión a un requisito y admite lo que no sabe con un plan para averiguarlo. Los ejercicios prácticos, de debugging y de diseño están en el [Apéndice A](15-Preguntas-y-Ejercicios-Apendice-A.md).
>
> **Conexiones:** aplica todo el material de [Módulo 01](01-Arquitectura-de-Software-Moderna.md) al [Módulo 14](14-System-Design-Interview.md). Complementa el [Módulo 16](16-Roadmap-y-Proyecto-Final.md), que convierte este banco en un plan de estudio y en el proyecto capstone.

---

## Introducción

### Cómo usar este banco

1. **Responde primero en voz alta** (con cronómetro), luego lee la respuesta modelo. El objetivo no es memorizar las respuestas: es entrenar tu **proceso**.
2. **Lee el nivel que te toca, pero responde con la barra senior.** Una pregunta junior bien contestada por un senior demuestra *profundidad sin pedantería*: contesta exactamente lo que preguntan y un nivel más, sin soltar un speech.
3. **Las respuestas modelo no son guiones**, son patrones de razonamiento. Adáptalas a tu experiencia real — en behavioral, tu historia importa más que el molde.

### La barra de evaluación 2026

La forma en que las empresas calibran un loop backend en 2026 se resume en **5 dimensiones** que se ponderan distinto según el nivel (fuente: guías de contratación 2026):

| Dimensión | Qué revela | Peso Mid | Peso Senior |
|---|---|---|---|
| **Fundamentals** | Estructuras de datos, concurrencia, HTTP, transacciones. ¿Razonan correctitud o solo recitan definiciones? | 30% | 15% |
| **System design** | ¿Dimensionan el problema, nombran trade-offs y defienden la decisión ante un follow-up no ensayado? | 25% | 35% |
| **Debugging under ambiguity** | La habilidad no guionada: un síntoma vago, sin causa obvia, y ves cómo lo acotan. | 25% | 20% |
| **Operational judgment** | Instinto de on-call: rollback vs fix-forward, blast radius, qué loguean, qué alarman. | 10% | 20% |
| **Tradeoff reasoning** | ¿Eligen y defienden, o se esconden detrás de "depende"? | 10% | 10% |

La celda que lo resume todo: **el peso del juicio operacional se duplica de mid a senior**. "Un mid puede crecer hasta el pager; un senior que nunca ha cargado un pager es un desarrollador con título de senior." En 2026, además: las **debugging rounds** (código roto, logs, tests que fallan) están creciendo frente a los puzzles de algoritmos, y las preguntas de **AI-infra** (sirviendo modelos en el hot path) son ya estándar.

### Calibración por nivel

- **Junior y early-career:** se evalúa *fundación y pendiente*: si razona, si se desatasca solo, y **si es honesto cuando no sabe**. Se tolera no saber; no se tolera inventar.
- **Mid-level:** ownership de una feature completa y una historia real de *shipped-and-broke* (algo que lanzaste y se rompió: qué aprendiste).
- **Senior:** el loop debe sentirse menos como un examen y más como *dos ingenieros discutiendo un diseño tomando café*. Se evalúa juicio; los datos se asumen. El patrón clásico: *"Diseña un rate limiter. Ahora te lo voy a romper"* — y el senior responde sin descomponerse.
- **Staff/Principal:** leverage (multiplicar equipos), arquitectura cross-team, y una **decisión que revertiste y lo que costó**.

---

# Parte 1 — Preguntas Junior (respondidas como senior)

> Una pregunta junior no merece una respuesta junior. El senior responde la pregunta exacta, y deja caer **un** nivel más de profundidad sin convertirla en conferencia.

**1. ¿Qué pasa cuando escribes una URL en el navegador?**

**Orientación:** Busca el flujo completo HTTP: DNS → TCP → TLS → request → response → render. Un senior lo recorre con orden, pero apunta dónde está la latencia real y las capas.

**Respuesta de un senior:** "El navegador resuelve el dominio por DNS (con caché local, luego recursivo), abre una conexión TCP, negocia TLS, envía un GET HTTP, recibe el HTML y los recursos, y los renderiza. En backend, lo que me importa son tres puntos: el **DNS** es la primera fuente de latencia (por eso los CDN y el TTL importan), el **TLS** añade dos RTTs (por eso HTTP/2 y conexiones reutilizadas), y el **time to first byte** depende de nuestra capacidad de servir desde el borde o con cache. Si el usuario está lejos del servidor, el mayor costo no es el cómputo: es el viaje de ida y vuelta."

**2. ¿Cuál es la diferencia entre GET y POST?**

**Orientación:** No basta con "GET lee y POST escribe". El senior distingue *semántica* (seguridad e idempotencia) de *implementación*, y menciona el resto de verbos.

**Respuesta de un senior:** "Semánticamente, GET es **seguro** (no cambia estado en el servidor) e **idempotente** (repetirlo N veces da lo mismo); POST no es ninguna de las dos — crea un recurso nuevo y repetirlo duplica. Eso tiene consecuencias prácticas: los GET se cachean, se pueden prefetchear y meter en URLs de libros de favoritos; los POST no. La regla operativa que aplico: si una acción cambia estado, no puede ser GET aunque sea 'leer de un carrito'. Y en APIs, el `DELETE` y el `PUT` también son idempotentes, lo que los hace seguros de reintentar con retries; el `POST` necesita una idempotency key."

**3. ¿Qué es una API REST?**

**Orientación:** El senior define los principios (recursos, verbos, stateless, códigos HTTP) pero advierte el matiz: REST es un estilo, y el purismo de Richardson Maturity Model rara vez se implementa completo.

**Respuesta de un senior:** "REST es un estilo arquitectónico sobre HTTP donde todo es un **recurso** identificado por URL, manipulado con verbos HTTP (GET/POST/PUT/PATCH/DELETE), sin estado de sesión en el servidor, y con códigos de estado expresivos. Lo importante para mí no es la liturgia: es que el diseño de la API comunique **semántica** — `DELETE /orders/{id}` es obvio, `POST /orders/{id}/cancel` también; `GET /orders?status=open` expresa una consulta de colección. El matiz senior: muy pocas APIs reales cumplen HATEOAS al 100% (el nivel 3 del modelo de Richardson); la mayoría son *HTTP JSON sobre recursos*, y está bien — el objetivo es contractos claros y evolucionables."

**4. ¿Qué es una base de datos relacional? ¿Clave primaria y foránea?**

**Orientación:** Fundamentos de modelado. El senior los explica con *por qué existen*: integridad referencial, unicidad, y cómo escalan.

**Respuesta de un senior:** "Una base de datos relacional organiza los datos en **tablas con relaciones** (one-to-many, many-to-many) y garantiza integridad transaccional (ACID). La **clave primaria** identifica de forma única cada fila y, por ser el índice natural, acelera el acceso por esa clave. La **clave foránea** referencia una primaria de otra tabla y es la que *mantiene* la relación: la base de datos puede impedir filas huérfanas o definir qué pasa al borrar (CASCADE, RESTRICT). El matiz que busco en un senior: la clave primaria define *cómo lees* (las PK de Postgres son índices B-tree), y las foráneas son caras de indexar cuando el volumen crece — por eso en el mundo NoSQL se renuncia a ellas a cambio de escala horizontal."

**5. ¿Qué es una transacción y qué es ACID?**

**Orientación:** Definición + por qué importa + cuándo ACID no escala (puente al mundo distribuido). Un senior no se queda en la definición.

**Respuesta de un senior:** "Una transacción agrupa operaciones para que sean **atómicas**: o se aplican todas o ninguna, incluso si el sistema se cae a mitad. ACID es el acrónimo: **Atomicidad** (todo o nada), **Consistencia** (las restricciones siempre se cumplen), **Aislamiento** (las transacciones concurrentes no se interfieren) y **Durabilidad** (lo confirmado sobrevive a un fallo). El uso típico: transferencia de dinero — dos UPDATEs que deben ir juntos. El matiz senior: en un solo nodo ACID es gratis; en sistemas distribuidos mantener ACID entre nodos es caro o imposible (ver CAP), y es cuando aparecen los patrones de consistencia eventual y sagas."

**6. ¿Qué diferencia hay entre array, lista enlazada y hash map?**

**Orientación:** Complejidad de operaciones (acceso, inserción, búsqueda) y uso en la práctica. El senior los compara en la misma tabla mental.

**Respuesta de un senior:** "Son tres contratos con costos distintos. **Array**: acceso por índice O(1), pero insertar/borrar en medio es O(n) por el desplazamiento. **Lista enlazada**: inserción/borrado en la cabeza O(1), pero acceso por índice O(n) — hoy casi no se usan en backend salvo en estructuras muy concretas. **Hash map**: búsqueda, inserción y borrado promedio O(1) gracias a la función hash, a costa de no tener orden. La decisión práctica: en backend, el 90% de los casos son *lookup por clave* → hash map; cuando necesito orden + rango → árbol (B-tree/red-black); y casi nunca listas enlazadas. Elegir la estructura correcta es decidir *cómo se accede*, antes que escribir código."

**7. ¿Qué es la complejidad Big O?**

**Orientación:** El senior la explica como *tasa de crecimiento*, no como "velocidad", y la conecta con datos reales (escala).

**Respuesta de un senior:** "Big O describe **cómo crece el tiempo o la memoria** de un algoritmo cuando la entrada crece, ignorando constantes. Es la herramienta para responder *'¿esto aguanta 100 millones de requests?'* sin ejecutarlo. O(1) no crece, O(log n) crece lentísimo, O(n) lineal, O(n log n) el típico de ordenar, O(n²) cuadrático — colapsa al doblar los datos. En backend, el momento en que importa es el **access pattern**: un lookup de hash map O(1) contra un JOIN mal indexado O(n·m) es la diferencia entre 1 ms y un timeout. Dato que uso siempre: para un millón de elementos, O(n) son ~microsegundos… O(n²) ya son segundos. La escala hace real lo teórico."

**8. ¿Qué es la orientación a objetos?**

**Orientación:** Encapsulación, herencia, polimorfismo — pero con el matiz senior de que el OOP es una herramienta, no un dogma, y cómo se manifiesta en la arquitectura (ver M01).

**Respuesta de un senior:** "Es un paradigma que organiza el código en **objetos** que combinan estado y comportamiento, con tres pilares: **encapsulación** (el estado se protege, se accede por métodos — la base de 'dependencias hacia dentro' de Clean Architecture), **herencia** (reuso de comportamiento, cada vez más criticada en favor de composición), y **polimorfismo** (un contrato, múltiples implementaciones — la base de los *ports & adapters*). El matiz senior: el valor real del OOP no es el paradigma en sí, es **gestionar la complejidad** acotando quién toca qué. Por eso en el mundo actual se prefiere composición sobre herencia y se combina con estilos funcionales donde el estado inmutable simplifica la concurrencia."

**9. ¿Qué es Git y para qué sirven `commit`, `branch` y `merge`?**

**Orientación:** El senior responde con el *flujo de trabajo* detrás de los comandos: el commit como unidad atómica revisable, la rama como aislamiento, el merge como integración — y conecta con CI/CD (M12).

**Respuesta de un senior:** "Git es un sistema de control de versiones **distribuido**: cada clon tiene la historia completa. El **commit** es una unidad atómica de cambio con mensaje y autor — su tamaño importa: commits pequeños y bien escritos hacen revisable la historia y permiten `git bisect` para encontrar qué cambio rompió algo. La **branch** es aislamiento de trabajo: puedes experimentar sin tocar lo estable. El **merge** integra ramas — y aquí el matiz: un merge limpio no es el objetivo, el objetivo es que la *integración sea continua* (trunk-based development + PRs pequeños que van a main), que es lo que hace que CI/CD funcione. El `git bisect` es mi herramienta favorita de debugging: encuentro el commit culpable en O(log n)."

**10. ¿Qué es un bug y cómo lo depurarías?**

**Orientación:** Pregunta trampa para ver proceso, no intuición. Un senior responde con el *método* de debugging, no con "miraría el código".

**Respuesta de un senior:** "Un bug es un desvío entre el comportamiento esperado y el observado, y la causa suele estar varios pasos antes del síntoma. Mi proceso es el del Pragmatic Programmer: primero **reproduzco** el fallo — si no lo reproduzco, no sé si lo arreglé; idealmente con un comando o un test que falle. Luego **formulo hipótesis** y las descarto con evidencia, no con corazonadas ('Don't Assume It — Prove It': si crees que una función funciona, demuéstralo con esos datos, en ese contexto). Cuando encuentro la causa, **arreglo la raíz, no el síntoma**, y añado el test que hubiera detectado el bug antes. Y lo hago sin pánico, sabiendo que el problema es mío aunque no sea mi culpa ('Fix the Problem, Not the Blame'). El rubber duck: explicar el código paso a paso en voz alta suele hacer que el bug salte solo."

**11. ¿Qué es un test unitario y uno de integración?**

**Orientación:** El senior distingue el *qué* (alcance) del *por qué* (velocidad y confianza) y el coste del test.

**Respuesta de un senior:** "Un **test unitario** aísla una unidad (una clase, una función) con sus dependencias mockeadas: corre en milisegundos, te dice *exactamente* dónde falla, y es la red de seguridad del día a día. Un **test de integración** verifica que componentes reales cooperan (servicio ↔ base de datos, servicio ↔ cola): más lento, más frágil, pero detecta lo que el unitario no puede — contratos, mapeos, SQL real. La regla del senior es la **pirámide de testing** y el test *hermético*: muchos unitarios (rápidos), menos de integración, pocos E2E. Y el test que falla primero es el que importa: en debugging, escribo el test que reproduce el bug *antes* de tocar el código (failing test before fixing code)."

**12. ¿Cómo manejas los errores en tu código?**

**Orientación:** Excepciones vs códigos, y la diferencia entre errores esperados e inesperados. El senior habla de *contratos* y observabilidad.

**Respuesta de un senior:** "Con una distinción mental clara entre **errores esperados** (que son parte del flujo: validación, no-encontrado, conflicto) y **excepciones** (lo inesperado: una base de datos que no responde). Los esperados se modelan como resultados explícitos (400/404/409, `Result` types, options); los inesperados se dejan propagar a un handler central que los loguea y devuelve un 500 genérico — *sin filtrar stack traces al cliente*. El matiz senior: el manejo de errores es un **contrato con el operador**: cada error que atrapo debe llegar a un log estructurado con contexto (qué request, qué recurso) para que el on-call pueda depurarlo sin reproducir el caso a mano. Silenciar excepciones es la deuda más cara que existe."

**13. ¿Qué es un índice de base de datos?**

**Orientación:** Estructura (B-tree), qué acelera, y el trade-off (writes más caros). El senior conecta con el plan de ejecución.

**Respuesta de un senior:** "Un índice es una estructura de datos auxiliar (típicamente un **B-tree**) que permite buscar por una columna sin escanear la tabla entera: pasa de O(n) a O(log n). Acelera los reads — filtros, JOINs, ORDER BY — a cambio de pagar en cada **write** (mantener el índice) y de ocupar espacio. El matiz senior: un índice no resuelve *la query*, resuelve **el access pattern**; por eso se decide por las consultas reales, no por cubrirlo todo. Los índices compuestos deben ordenar sus columnas según la consulta (la más selectiva o la de igualdad primero), y un **covering index** (incluye las columnas del SELECT) elimina el lookup a la tabla. Siempre valido con `EXPLAIN` — los índices que no se usan son solo costo."

**14. ¿Qué es una caché?**

**Orientación:** Definición + el trade-off fundamental (frescura vs velocidad) + invalidación. Un senior no responde solo "una caché guarda datos".

**Respuesta de un senior:** "Una caché guarda una **copia** de datos costosos de calcular en un lugar rápido, para servir lecturas sin tocar el origen. El trade-off fundamental es **frescura vs velocidad**: acepto que el dato pueda estar desactualizado a cambio de latencia y de no golpear la base de datos. Lo importante no es *qué* es, es *cómo se invalida*: cache-aside (la app llena y TTLs), write-through/write-behind, y el problema clásico del **cache stampede** cuando caduca todo a la vez. Decisión senior: nunca cacheo lo que no se lee repetidamente, y defino el TTL según cuánta eventualidad tolera el negocio — el catálogo de productos aguanta 60s de desfase; un saldo no aguanta ni uno."

**15. ¿Qué es un hilo y qué es un proceso?**

**Orientación:** El senior explica el modelo de concurrencia y lo conecta con el mundo real (cuántos hilos crea un servidor web, los "costly threads").

**Respuesta de un senior:** "Un **proceso** es una instancia de programa con su propio espacio de memoria aislado; un **hilo** es una unidad de ejecución *dentro* de un proceso que comparte su memoria. Los hilos son baratos de crear pero comparten estado — de ahí los race conditions y la necesidad de locks. El matiz práctico: en un servidor web, el modelo tradicional es *un hilo por conexión*, que colapsa a ~10K hilos (el problema C10K); por eso los runtimes modernos usan **event loops** y E/S asíncrona (Node, Go con goroutines, Erlang con procesos livianos de ~2KB) que gestionan cientos de miles de conexiones con pocos hilos del SO. Elegir el modelo de concurrencia correcto es decidir cuántas conexiones aguantas por máquina."

**16. ¿Qué son los códigos de estado HTTP y cuáles usas más?**

**Orientación:** El senior los agrupa por familia y da el *uso correcto* de los importantes — señal de senior: usar 409 vs 422 vs 400 con criterio.

**Respuesta de un senior:** "Son la semántica de la respuesta. **2xx**: 200 OK, 201 Created (POST), 204 No Content. **3xx**: redirecciones — 304 Not Modified (para caching condicional con ETags). **4xx**: errores del cliente — 400 (mala petición), 401 (no autenticado), 403 (no autorizado), 404, 409 (conflicto, ej: recurso ya existe), 422 (validación semántica), 429 (rate limit). **5xx**: errores del servidor — 500, 502 (bad gateway), 503 (no disponible), 504 (timeout de upstream). El detalle senior: el 4xx/5xx bien usado es un **contrato de cliente**: 4xx no debe reintentarse ciegamente, 5xx sí (con cuidado). Y el `Retry-After` en un 429 es cortesía que salva tu propia infraestructura de reintentos salvajes."

**17. ¿Cómo funciona el DNS?**

**Orientación:** Jerarquía (recursivo → raíz → TLD → autoritativo) y el punto operativo: caché y TTL. El senior conecta con latencia y failover.

**Respuesta de un senior:** "El DNS traduce nombres a IPs mediante una **jerarquía**: un resolver recursivo (tu ISP o 8.8.8.8) consulta a los servidores de raíz, luego a los de TLD (`.com`), luego al **autoritativo** del dominio, que devuelve la IP. La clave operativa es la **caché con TTL**: cada nivel cachea la respuesta durante el TTL, así el nombre se resuelve en microsegundos tras la primera vez. El matiz senior: el DNS es una herramienta de *direccionamiento y failover* — cambiar el TTL a bajo antes de un failover de región permite redirigir tráfico rápido; y los **DNS-based load balancing** (como hacen los CDN) resuelven a la IP del edge más cercano. Una mala respuesta DNS se convierte en la peor latencia que existe."

**18. ¿Qué es la integración continua (CI)?**

**Orientación:** El senior la explica como *frecuencia de integración* y la conecta con su propósito: detectar conflictos temprano. (M12 lo profundiza.)

**Respuesta de un senior:** "CI es la práctica de integrar el código a main **varias veces al día** en lugar de en ramas largas, con cada integración validada automáticamente (build + tests + análisis estático). El objetivo no es 'tener un pipeline': es que el *conflicto de integración* se detecte en minutos, cuando el código que lo causa está fresco y el costo de arreglarlo es mínimo — el 'mapeo del costo' de integrar temprano contra integrar tarde. Un senior sabe que la CI solo funciona con commits pequeños y frecuentes (trunk-based) y con tests rápidos y fiables; si la CI tarda 40 minutos o da falsos fallos, el equipo la esquiva y pierde su valor."

---

# Parte 2 — Preguntas Mid (respondidas como senior)

> Aquí se evalúa *ownership*: que hayas llevado features reales a producción y entendido sus consecuencias. El senior responde con el *por qué* y los *trade-offs*, no con el glosario.

**1. ¿Qué es una race condition y cómo la evitas?**

**Orientación:** El senior define el problema de concurrencia, da ejemplos reales (carrito, inventario, contadores) y las estrategias (atómico, locking, versionado).

**Respuesta de un senior:** "Una race condition ocurre cuando dos operaciones concurrentes leen-modifican-escriben el mismo dato sin sincronizar, y el resultado depende del orden. El clásico: dos cargos simultáneos sobre el mismo saldo, o un inventario que se vende dos veces. Las estrategias van de menos a más caras: **operaciones atómicas** (un `INCR` de Redis, un `UPDATE SET stock = stock - 1 WHERE stock >= 1` que es atómico en SQL), **locking optimista** (un campo `version`: el UPDATE falla si la versión cambió) y **locking pesimista** (`SELECT FOR UPDATE`). La elección senior: depende del ratio de contención. Para contadores y stock, el conditional update basta; para pagos, uso transacciones con verificación + constraint único; y siempre pregunto *'¿quién más escribe esto y con qué garantía?'* antes de elegir."

**2. Explica los niveles de aislamiento de transacciones.**

**Orientación:** Read uncommitted → serializable, con los fenómenos (dirty read, non-repeatable, phantom) y el trade-off con rendimiento. Un senior usa ejemplos.

**Respuesta de un senior:** "Definen cuánta interferencia toleran las transacciones concurrentes. **Read uncommitted**: veo cambios no confirmados (dirty reads) — casi nunca se usa. **Read committed** (por defecto en Postgres): no veo dirty reads, pero sí non-repeatable reads (el mismo SELECT devuelve distinto dentro de la transacción). **Repeatable read / snapshot isolation**: cada transacción ve una foto consistente del inicio; sigue el problema de **write skew** (dos transacciones leen lo mismo y escriben en base a una premisa obsoleta). **Serializable**: resultado idéntico a ejecutarlas en serie — mata write skew y phantoms a costa de rendimiento o de abortos (SSI). La decisión: subir de aislamiento cuesta rendimiento; subo solo en los paths donde la correctitud lo exige (pagos, inventario). Para detectar write skew, `SELECT ... FOR UPDATE` en el *read* es la herramienta."

**3. ¿Qué es el problema N+1 y cómo lo resuelves?**

**Orientación:** ORMs y queries. El senior lo detecta, lo mide (EXPLAIN/logs), y da las soluciones (JOIN, batch, eager load).

**Respuesta de un senior:** "El N+1 es cuando para N registros hago 1 query + N queries (una por registro): típico en ORMs que cargan lazy las relaciones — listo 100 pedidos y hago 100 queries más para los clientes. El costo no es solo el tiempo: son los round trips y la presión en la base de datos. Se detecta mirando los logs de queries o el `EXPLAIN`. Soluciones: **JOIN** en la query (una sola ida), **batch/eager loading** en el ORM (`fetch join` de JPA, `selectin`), o **denormalizar** cuando el patrón es sistemático (guardar el nombre del cliente en el pedido). El matiz senior: el N+1 es un *síntoma* de modelar según el ORM y no según el access pattern; la pregunta correcta es '¿cuántas queries hace esta página?' — y el número debe ser constante, no proporcional a los datos."

**4. ¿Cómo diseñarías un índice para una tabla con millones de filas?**

**Orientación:** Access patterns → columnas → orden → cubrir. El senior piensa en las queries reales y en el costo de los writes.

**Respuesta de un senior:** "Primero aclaro los **access patterns**: las queries reales y sus filtros. Para `WHERE user_id = ? AND created_at > ? ORDER BY created_at`, un **índice compuesto `(user_id, created_at)`** sirve el rango y el orden; las columnas de igualdad van antes que las de rango. Si el SELECT solo pide esas columnas, lo convierto en **covering index** para evitar el lookup a la tabla. Considero el **cardinality**: un índice sobre una columna con 3 valores distintos casi no ayuda. Y el trade-off: cada índice extra hace los writes más caros — con una tabla de millones de filas y write-heavy, un índice innecesario se paga todos los días. Verifico con `EXPLAIN (ANALYZE)`: si el plan no lo usa, el índice sobra. En tablas enormes, quizá la respuesta correcta sea particionar o pasar a un almacén NoSQL — el índice no arregla un access pattern mal resuelto."

**5. ¿Qué es cache-aside y cómo manejas la invalidación?**

**Orientación:** El senior explica el patrón, el problema de la doble escritura (DB+cache), y la respuesta: TTLs, invalidación en escritura, o dejar de invalidar.

**Respuesta de un senior:** "Cache-aside es: la aplicación consulta la caché; si hay miss, lee de la base, guarda en caché y responde. El problema que todo el mundo subestima es la **invalidación**: cuando escribo en la base, ¿cómo refresco la caché? Las opciones son: **TTL** (el dato expira solo — simple, pero sirves desfase), **invalidación en escritura** (borrar la clave al escribir — pero si el write falla tras borrar, tienes un miss que recalcula, que es peor para la DB), y el patrón correcto: **escribir a la base, y eliminar la clave de caché** (delete, no update) para que el próximo read la repoble — porque actualizar la caché a mano genera carreras entre writers. El matiz senior: el cache stampede (muchos misses a la vez tras una invalidación masiva) se mitiga con TTL escalonado y *single flight*. Y la regla: la base de datos es la fuente de verdad; la caché es siempre descartable."

**6. ¿Cómo versionas una API pública?**

**Orientación:** El senior distingue versionado (URL vs header) de *compatibilidad* (aditivo vs breaking), y cuándo cada uno.

**Respuesta de un senior:** "Primero separo dos cosas: **versionar el contrato** y **mantener compatibilidad**. Para versionar: **URL (`/v1/orders`)** es simple, explícito y cachea bien — el estándar para APIs públicas; **header (`Accept: application/vnd.api+json;version=2`)** mantiene las URLs limpias pero oculta la versión y complica la depuración. Mi elección: URL para versiones principales, y prefiero **evolucionar aditivamente** (añadir campos/endpoints, nunca quitarlos, con defaults que no rompan) — así la mayoría de cambios no requieren bump de versión. Un cambio breaking (cambiar un tipo, quitar un campo) exige versión nueva y **deprecación con fechas**: anuncio, mantengo la vieja un período, mido tráfico, y solo entonces la apago. El matiz senior: si tu equipo versiona cada cambio, el problema no es el versionado, es que no estás diseñando contratos evolutivos."

**7. ¿Qué es la idempotencia y por qué importa en APIs?**

**Orientación:** Definición + el problema que resuelve (retries) + implementación (idempotency keys + constraint). El senior lo conecta con pagos (M14).

**Respuesta de un senior:** "Idempotencia: ejecutar la operación N veces produce el mismo resultado que una vez. Importa porque **la red reintenta**: un timeout del cliente no significa que el servidor no procesara la request, y sin idempotencia el retry duplica (cobrar dos veces, crear dos pedidos). `GET`, `PUT`, `DELETE` son naturalmente idempotentes; `POST` no lo es. La solución para `POST` es la **idempotency key**: el cliente envía una clave única (Stripe lo popularizó); el servidor guarda el resultado indexado por esa clave y devuelve el mismo resultado ante repeticiones, con un **constraint único** en la base para que dos requests concurrentes no se cuelen. Implementación: en el `POST /payments`, guardo `(idempotency_key, resultado)` con unique constraint, y la transición del estado se protege para que solo una request pase de PENDING a CHARGING."

**8. ¿Qué es rate limiting y cómo lo implementas?**

**Orientación:** Algoritmos (token bucket, sliding window), dónde (edge vs servicio), y el matiz senior: ¿fail open o fail closed? (el clásico "then I'll break it").

**Respuesta de un senior:** "Rate limiting controla cuántas requests acepta un cliente en una ventana de tiempo. Algoritmos: **token bucket** (admite ráfagas, simple, el de API Gateway), **fixed window** (barato pero con el problema de los bordes: 100 requests justo antes y justo después de la frontera = 200), **sliding window log/counter** (preciso, más memoria). Decisión de *dónde*: en el **edge** (API Gateway/WAF) protege todo el sistema; en el **servicio** permite lógica por plan. Y la pregunta senior que casi nadie responde sola: **¿qué pasa si el Redis que cuenta se cae?** Fail open (dejas pasar todo — proteges la experiencia pero pierdes el control) o fail closed (cortas a todos — te protege pero puede bloquear a clientes pagos). La respuesta correcta es *depende del riesgo*: para un login, fail closed; para un feed, fail open. No hay respuesta limpia, y nombrarlo es el punto."

**9. ¿Cómo descompondrías un monolito en microservicios?**

**Orientación:** El senior advierte el antipatrón (descomponer por capas técnicas) y defiende los bounded contexts (M02), el data ownership y el costo de la red.

**Respuesta de un senior:** "Con un aviso previo: no descompongas si no hay una razón — un monolito modular bien hecho muchas veces es la respuesta correcta. Si descompones, la unidad es el **bounded context** (DDD): servicios alrededor de capacidades de negocio, no de capas técnicas (nunca 'el servicio de servicios'). Cada servicio **es dueño de sus datos** — la base de datos compartida es el mayor antipatrón. Reglas que aplico: los límites deben reducir el *acoplamiento* (qué cambia junto queda dentro), la comunicación entre servicios va por **contratos y eventos** (no por llamadas síncronas encadenadas), y el coste operativo (red, observabilidad, sagas, despliegues independientes) se paga **antes** de ver el beneficio. La estrategia de corte: *strangler fig* — extraer un servicio a la vez por el camino de menor fricción, con los datos migrados junto al servicio."

**10. ¿Qué es at-least-once delivery y cómo lidias con los duplicados?**

**Orientación:** Semánticas de entrega + la respuesta: idempotencia del consumidor (M03). El senior lo conecta con exactly-once y Kafka/SQS.

**Respuesta de un senior:** "At-least-once significa que un mensaje se entrega *una o más veces*: no se pierde, pero puede duplicarse. Es la semántica por defecto en SQS y Kafka (según configuración) porque viene de la naturaleza de los ACKs: si el consumidor muere tras procesar pero antes de ACK, el broker redelivers. La única defensa correcta es un **consumidor idempotente**: el procesamiento debe ser repetible sin efecto doble — mediante IDs de mensaje con deduplicación, operaciones idempotentes (upserts, conditional writes) o la combinación que llamamos *effectively-once*. El matiz senior: *exactly-once real de extremo a extremo no existe* en sistemas distribuidos (es el problema de los dos generales); lo que existe es *at-least-once + idempotencia*, que se comporta como exactly-once. Por eso pago siempre el costo de la idempotencia y nunca confío en 'el broker no duplica'."

**11. ¿Qué es el patrón transactional outbox?**

**Orientación:** El problema del dual-write y la solución (M03). El senior lo explica con el caso de pago + evento.

**Respuesta de un senior:** "El outbox resuelve el **dual-write problem**: cuando actualizo la base de datos *y* publico un evento (a Kafka/SQS), si hago las dos cosas en pasos separados y el segundo falla, el sistema queda inconsistente — el pedido se guardó pero el evento nunca se emitió, o al revés. La solución: escribir el evento en una tabla **outbox dentro de la misma transacción** que el cambio de estado; luego un relay (CDC con Debezium, o un proceso que hace polling) publica los eventos pendientes al bus. La garantía resultante es **at-least-once**, así que los consumidores siguen necesitando idempotencia. Es el patrón que sostiene 'saga + eventos' en pagos: el cargo y el PaymentIntent en el bus salen atómicamente. Cuando alguien dice 'publico el evento después de guardar', yo digo '¿y si se cae entre medio?' — y esa es la respuesta del outbox."

**12. Explica el teorema CAP en la práctica.**

**Orientación:** No la definición de libro: el matiz de que C y A solo chocan *bajo partición* (M05, M14). El senior lo aplica por componente.

**Respuesta de un senior:** "CAP dice que bajo una **partición de red** hay que elegir entre consistencia (todos ven lo mismo) y disponibilidad (todos reciben respuesta). El error más común es creer que C y A chocan siempre: **en operación normal puedes tener ambas** — chocan solo durante la partición. Por eso lo completo con **PACELC**: en normalidad (E), se elige entre latencia (L) y consistencia (C). En la práctica, decido **por componente**: un feed elige AP durante la partición (responde con datos quizá viejos, nunca se cae) y EL en normalidad (la consistencia eventual es aceptable); un ledger financiero elige CP (no acepta respuestas que puedan ser incorrectas) y paga latencia. Un senior no responde 'elijo AP': responde *qué componente*, *qué garantía*, y *qué acepta el negocio*."

**13. ¿Qué es la replicación y qué problemas trae?**

**Orientación:** Leader/follower, réplicas síncronas/asíncronas, y los problemas de lag (read-your-writes, monotonic reads). El senior conecta con cuando el lag rompe UX (M10).

**Respuesta de un senior:** "Replicar es mantener copias del mismo dato en varios nodos para disponibilidad, durabilidad y lecturas escalables. El modelo común es **single-leader**: todas las escrituras van al primario; los seguidores aplican el log y sirven lecturas. El trade-off: **asíncrono** (rápido, pero el seguidor tiene *lag* — puedes leer un dato viejo) vs **síncrono** (consistente, pero si el seguidor no responde, las escrituras se bloquean). El lag no es teórico: rompe el *read-your-writes* (publicas y al refrescar no ves tu post — se lee del seguidor) y el *monotonic read* (recargas y el dato 'vuelve atrás'). Mitigaciones: leer las propias escrituras desde el primario o con consistencia a nivel de sesión, y pensar el lag como una deuda que pagas en UX. La decisión senior: replica para *disponibilidad y escala de lecturas*, y diseña para tolerar lag donde el negocio lo permita."

**14. ¿Cómo autenticas y autorizas una API?**

**Orientación:** El senior separa autenticación (quién eres) de autorización (qué puedes), compara session vs JWT vs OAuth, y advierte los problemas de seguridad reales (M08).

**Respuesta de un senior:** "Dos problemas distintos: **autenticación** (¿quién es?) y **autorización** (¿puede hacer esto?). Para APIs modernas: **OAuth 2.1 / OIDC** para delegar login (Authorization Code + PKCE para apps), y el acceso al API con **JWT firmado** (access tokens) o sesiones server-side. La elección JWT vs session es un trade-off real: JWT es stateless (el API no consulta al identity provider en cada request, escalas gratis) pero no se invalida hasta que expira (no puedes matar un token robado, salvo con allowlist/denylist o tokens cortos + refresh); sesiones son revocables al instante pero cada request consulta el store (Redis) y paga latencia. Mi práctica: tokens de acceso **cortos** (5-15 min) + refresh tokens con rotación, scopes granulares, y siempre HTTPS + secreto fuera del código. La autorización se valida en cada servicio (no solo en el gateway) — nunca confíes solo en el API Gateway."

**15. ¿Qué es un despliegue blue-green?**

**Orientación:** Definición + el beneficio real (rollback instantáneo) + el problema: la base de datos (M12). El senior explica cuándo sí y cuándo no.

**Respuesta de un senior:** "Blue-green mantiene dos entornos de producción idénticos: el tráfico va a uno (blue) mientras el otro (green) recibe la versión nueva; al validar, se cambia el router al verde. El beneficio es el **rollback instantáneo**: volver es re-dirigir el tráfico, no re-desplegar. El problema que todos subestiman es la **base de datos compartida**: los dos entornos usan la misma BD, así que el esquema debe ser *compatible hacia adelante* — los cambios de esquema son aditivos (nuevas columnas nullable) y el código viejo debe funcionar con el esquema nuevo. Si una migración es destructiva, el blue-green no te salva del rollback. Mi regla: blue-green para servicios stateless con DB compartida aditiva; y combinar con feature flags para no acoplar *deploy* y *release*."

**16. ¿Qué es la observabilidad y en qué se diferencia del monitoring?**

**Orientación:** Los tres pilares (métricas, logs, traces) + la diferencia: monitoring te dice qué falla, observabilidad te deja descubrir por qué (M07).

**Respuesta de un senior:** "**Monitoring** te dice que algo falla: dashboards y alarmas sobre problemas *conocidos*. **Observabilidad** te permite investigar *por qué* falla algo que nunca habías visto — se basa en los **tres pilares**: métricas (números agregados: RED — Rate, Errors, Duration), logs (eventos con contexto), y **distributed tracing** (el viaje de una request por todos los servicios, con trace ID en los headers). La diferencia práctica: el monitoring responde preguntas que ya sabías hacer; la observabilidad te deja hacer preguntas nuevas bajo incertidumbre. Mi regla: cada servicio expone métricas RED, logs estructurados con correlation IDs, y trazas de extremo a extremo; y los SLIs se traducen a **SLOs** — no a alarmas arbitrarias. Un sistema sin observabilidad es una caja negra: en una entrevista senior, la observabilidad es parte del diseño, no un extra."

**17. ¿Qué es sharding y qué problemas trae?**

**Orientación:** Particionar datos (M05, M10), la elección de la shard key, hot keys, y rebalanceo. El senior da las reglas.

**Respuesta de un senior:** "Sharding es partir los datos en varios nodos para que ninguno sea el cuello de botella — horizontal scaling real. La **shard key** lo decide todo: debe distribuir el acceso de forma uniforme (por eso se evitan claves con pocos valores o muy sesgadas) y servir los access patterns principales (si todo consulta por user_id, shard por user_id). Los problemas: **hot keys** (una clave que concentra tráfico — un usuario enorme, una celda geográfica en Times Square), consultas que cruzan shards (deben ser evitadas o hechas en un nodo coordinador), y el **rebalanceo** cuando añades nodos (por eso se usa **consistent hashing**: añadir/quitar nodos solo mueve ~1/N de las claves). El matiz senior: sharding es de las últimas palancas que tiro — antes cacheo, particiono por tiempo, o separo lecturas; porque una vez shardeado, cada query y cada migración se complican para siempre."

**18. ¿Qué es un circuit breaker y cuándo lo usas?**

**Orientación:** El patrón de resiliencia (M10) con sus estados y el *por qué*: proteger a los llamadores de un servicio degradado.

**Respuesta de un senior:** "El circuit breaker envuelve llamadas a una dependencia externa: mientras los fallos superan un umbral, el circuito **abre** y las llamadas fallan *inmediatamente* sin intentar la conexión, protegiendo a los llamadores de agotar sus hilos/recursos contra un servicio caído. Estados: **cerrado** (normal, se cuentan fallos) → **abierto** (rechazo instantáneo, da tiempo a recuperarse) → **semi-abierto** (tras un timeout deja pasar unas pruebas; si funcionan, cierra). El matiz senior que pocos mencionan: hay que definir **qué pasa cuando abre** — el fallback. Y la pregunta de diseño real: *¿cómo pruebas el fallback en staging si no puedes tumbar el servicio de producción?* — con pruebas de caos, toggles y simulación de fallos. El circuit breaker se usa contra dependencias lentas/frágiles (un API externo, una BD en problemas), nunca como excusa para no arreglar la causa."

---

# Parte 3 — Preguntas Senior (respondidas como senior)

> Aquí el loop es *juicio*, no quiz. Los datos se asumen; se evalúa cómo razonas bajo ambigüedad, cómo decides con información incompleta, y cómo conduces el incidente. El patrón "then I'll break it" es el estándar: el entrevistador acepta tu diseño y lo ataca.

**1. "Diseña un rate limiter. Ahora te lo voy a romper: ¿qué pasa si el Redis se cae?"**

**Orientación:** El clásico "then I'll break it". El senior expone las decisiones (dónde, unidad, algoritmo) y *anticipa* el modo de fallo antes de que lo pregunte, con la disyuntiva fail open/fail closed.

**Respuesta de un senior:** "Mi diseño: rate limiting en el **edge** (API Gateway) con **sliding window** en Redis, clave por API key con límites por plan. Antes de que me lo rompas, te adelanto el modo de fallo que me preocupa: el Redis es un *single point of failure* del control. Si se cae tengo dos opciones: **fail open** — dejo pasar todo (protejo la experiencia y los clientes pagos, pero pierdo el control: el abuso entra) — o **fail closed** — corto todo (protejo el sistema, pero puedo bloquear a un cliente legítimo, que es un fallo de producto). No hay respuesta limpia: depende del riesgo. Para login/anti-fraude elijo fail closed (un ataque de fuerza bruta es peor que un falso 429); para un feed público elijo fail open (la disponibilidad es el producto). Además: réplica del contador en memoria local como fallback, y alerta inmediata si la divergencia entre Redis y local se dispara. Y el *crash*: después de un corte, todos los clientes reintentan → backoff con jitter o el edge mismo se convierte en la estampida."

**2. "Cuéntame un incidente que hayas manejado de principio a fin."**

**Orientación:** La pregunta de *operational judgment*. El senior estructura con el proceso de incidente (detección → triage → mitigación → postmortem) y demuestra *calma y método*, no heroísmo.

**Respuesta de un senior:** "Te cuento uno de pago que me tocó. **Detección:** una alerta de rate de éxito de pagos cayendo — no el dashboard de uptime, la métrica de negocio. **Triage:** en 5 minutos separé lo crítico de lo interesante — las *escrituras* del ledger fallaban, no el API; eso acotaba la zona. **Hipótesis:** el primario de la BD de pagos se había degradado. Lo confirmé con métricas de latencia y el backlog de la cola creciendo — no adivinando. **Mitigación:** decidí **fix-forward** en vez de rollback (la migración de esquema era aditiva pero el código nuevo dependía de ella), hice failover al nodo de respaldo y el tráfico se estabilizó en minutos; los mensajes de la cola siguieron ahí (la cola es el buffer, nada se perdió). **Postmortem:** 5 whys — la causa raíz no fue el nodo, fue que el *auto-scaling del storage* no tenía umbral de alerta; lo arreglamos con la alerta, y añadimos el test de integración que no había. Lo importante del relato: proceso reproducible, decisiones con evidencia, y terminar en prevención — no en 'lo arreglamos y ya'."

**3. "¿Cómo decidirías el modelo de consistencia de cada componente de un sistema?"**

**Orientación:** Trade-off reasoning por componente (M05/M14). El senior no da una respuesta global: distingue *correctness-first* de *eventual-ok* y el caso gris read-your-writes.

**Respuesta de un senior:** "Nunca decido un modelo de consistencia 'para el sistema': lo decido **por componente**, según lo que el negocio tolera. Regla de oro: los datos de correctitud exigen **strong consistency** — pagos, inventario, balances, unicidad — y en partición eligen consistencia sobre disponibilidad (CP), pagando latencia. Los datos sociales toleran **eventual consistency** — feed, presencia, likes, ubicación — y en partición eligen disponibilidad (AP). El caso gris que la gente olvida es el **read-your-writes**: si un usuario postea y su feed no muestra su propio post al refrescar, la confianza se rompe aunque 'eventualmente' converja. Se resuelve sin strong consistency global: lecturas desde el primario para lo propio, o consistencia a nivel de sesión (afinidad). El último criterio: coste. La consistencia fuerte cuesta latencia y disponibilidad; si el negocio no la necesita, es deuda pagar por ella. La decisión se justifica contra un requisito, no contra un gusto."

**4. "¿Qué haces cuando una dependencia externa empieza a fallar en producción?"**

**Orientación:** Resiliencia + juicio operativo: retry con backoff, circuit breaker, degradación, DLQ, y *nunca* dejar que el externo caído bloquee el sistema (M10).

**Respuesta de un senior:** "Primero, **limito el daño**: el circuit breaker abre para que los llamadores no se agoten intentando, y cada path degradado tiene un **fallback definido** (servir datos de cache, responder con una degradación útil, o una cola). Los retries llevan **backoff exponencial + jitter** — si millones de clientes reintentan a la vez, la dependencia caída empeora (thundering herd). Luego **mitigo el tráfico**: si es una API de terceros, reduzco el ritmo; si es nuestra, investigo con la observabilidad mientras el breaker protege. Los mensajes que no se procesan van a una **DLQ** para reprocesar después, nunca se pierden. Y una regla que aplico siempre: **aislar por canal** — en notificaciones, que un outage de email no afecte al push; en general, que la dependencia frágil tenga su propio budget de latencia y su propio fallback. Al final, postmortem: por qué la dependencia falló y, sobre todo, por qué *nosotros* no lo detectamos antes."

**5. "Tienes que reducir a la mitad el coste de un sistema de streaming. ¿Dónde miras?"**

**Orientación:** Cost reasoning (rubric 2026). El senior identifica la palanca dominante: en streaming, el ancho de banda; y va de lo caro a lo barato con números.

**Respuesta de un senior:** "Empiezo por encontrar **dónde está el costo dominante**, porque una mejora del 1% ahí vale más que el 50% en un componente pequeño. En streaming, el costo dominante es el **ancho de banda** (terabits por segundo), no el cómputo. Palancas en orden: (1) **compresión** — mejorar la eficiencia del codec es la más alta: 1% de compresión ahorra millones al año; (2) **tasa de cache hit del CDN** — cada byte servido desde el edge cuesta una fracción del que viene del origen: subir el hit ratio de 95% a 98% mueve el 60% del tráfico de origen; (3) **el ladder de ABR** — no codificar bitrates que nadie usa, y que el cliente no pida más de lo que su red permite (un cliente que pide 4K en 3G es puro desperdicio); (4) **acuerdos de peering** — en el Open Connect, dónde se colocan las caches. La respuesta que buscan: *nombro la palanca dominante y cuantifico* — no optimizo el CPU del codificador porque eso no es donde está el dinero. Y cierro con la medición: si no sé el costo por byte servido, no puedo decidir nada."

**6. "Cuéntame una decisión de arquitectura que tuviste que revertir."**

**Orientación:** Pregunta de *staff/senior*: honestidad + lo que costó + qué aprendiste. El senior muestra juicio al reconocer el error temprano.

**Respuesta de un senior:** "Elegí event sourcing para un sistema de facturación porque el marketing de la inmutable audit trail era irresistible. Lo descubrimos mal: el equipo no tenía la disciplina para manejar la evolución del esquema de eventos, cada cambio de regla de negocio era una migración de eventos compleja, y el modelo mental no encajaba con la feature real (que era un CRUD con estados). Revertimos a un modelo relacional clásico cuando la deuda empezó a superar al beneficio — fue costoso: el código de las proyecciones y las migraciones se fue, y perdimos semanas. Lo que aprendí: el *patrón* debe justificarse por el access pattern, no por la elegancia; y una decisión arquitectónica grande se valida con un **spike acotado** (2 semanas) antes de comprometer todo el equipo. El error no fue elegir event sourcing; fue no haber exigido la prueba de que encajaba. Reconocer el error a tiempo fue la decisión correcta aunque doliera."

**7. "¿Cómo escalas un sistema sin datos concretos? Hazme las estimaciones."**

**Orientación:** El senior hace supuestos explícitos y calcula con potencias de diez (10⁵ segundos/día), mostrando que *el proceso* es lo evaluado, no la cifra.

**Respuesta de un senior:** "Sin datos, **declaro supuestos** y calculo en voz alta. Asumo 10M DAU (público de una app nacional), cada uno hace 10 requests/día al API → 100M requests/día. Con 10⁵ segundos por día, son ~1.000 QPS promedio; con factor de pico 5×, **~5.000 QPS pico** — trivial para un API stateless escalado horizontalmente, el problema no está ahí. Almacenamiento: si cada request genera un registro de 1 KB, son 100 GB/día, ~36 TB/año — ahí ya pienso en particionar por tiempo y en retención. Lecturas: si es read-heavy 10:1, necesito **cache** y quizá CDN. El punto: la estimación no es para acertar el número — es para que **cada decisión del diseño tenga un número detrás** (¿necesito cache? sí, porque 100:1 de reads; ¿necesito sharding? no, hasta ~1M QPS o varios TB). Si los supuestos cambian, el diseño cambia; lo importante es que el *razonamiento* sea visible y defendible."

**8. "¿Cuándo es una mala idea usar microservicios?"**

**Orientación:** El senior defiende el monolito modular y enumera las señales de que la descomposición es deuda: no hay límites claros, el equipo es pequeño, la consistencia transaccional importa (M02).

**Respuesta de un senior:** "Microservicios son malos por defecto — se justifican por un requisito, no por moda. Malas ideas concretas: (1) **sin bounded contexts claros**: si no ves dónde cortar, cada 'servicio' será un monolito acoplado distribuido — lo peor de ambos mundos; (2) **equipo pequeño**: un equipo de 5-10 personas no gana nada con la independencia de despliegue y paga todo el coste operativo (red, observabilidad, sagas); (3) **transacciones que cruzan servicios**: si el flujo core necesita ACID entre lo que vas a separar, descomponer te obliga a sagas y consistencia eventual — y a veces la correctitud no lo tolera; (4) **data ownership compartido**: si van a compartir la base de datos, no son microservicios. Mi regla: empiezo por **monolito modular** con límites bien marcados (módulos por bounded context), y extraigo servicios solo cuando un equipo lo exige (escala de despliegue independiente, equipos autónomos). La deuda de la descomposición se paga antes de ver el beneficio."

**9. "¿Qué pasa si un endpoint empieza a devolver timeouts de forma intermitente? ¿Cómo lo investigas?"**

**Orientación:** Debugging under ambiguity (la dimensión no guionada). El senior reproduce, acota, formula hipótesis con evidencia y arregla la raíz.

**Respuesta de un senior:** "Lo primero: **reproducir y acotar**. ¿Intermitente para quién — un usuario, una región, un porcentaje del tráfico? Si es un subset, acoto: ¿peticiones concretas, payloads grandes, clientes específicos? Busco patrones en las métricas: **p99 vs p50** — si el p50 está bien y el p99 disparado, hay una cola o un hot spot (una clave caliente, un lock, GC); si todo sube, es capacidad o una dependencia. Hipótesis con evidencia, no corazonadas: miro el tracing distribuido del endpoint — ¿dónde se queda la request? ¿en nuestra lógica, en la BD, en una llamada externa? El sospechoso clásico: un **lock contención** o un **connection pool agotado** (threads esperando) — el síntoma de timeout con el p50 sano. Confirmo con métricas del pool y del lock, no adivinando. Arreglo la raíz, añado la alerta que debía existir, y dejo el test que hubiera detectado la regresión. El orden importa: primero entiendo, luego toco."

**10. "¿Cómo sirves una feature de IA en un hot path sin arruinar la latencia?"**

**Orientación:** AI-infra (2026 estándar). El senior piensa en latencia budget, degradación, cacheo, colas y *fallback cuando el modelo no responde*.

**Respuesta de un senior:** "Primero defino el **latency budget**: el hot path de un feed tiene 300ms de presupuesto; un LLM en línea cuesta 1-3 segundos — así que la pregunta es *¿qué parte de la feature puede ser async?* Clasifico: lo **síncrono crítico** (una clasificación, un re-rank corto) va con modelos pequeños/baratos o embeddings cacheados; lo **pesado** (generación, resumen) va **fuera del hot path**: se encola, el usuario recibe la respuesta cuando esté lista (patrón de resultado async + webhook/polling). Luego **caché**: la mayoría de las consultas se repiten (mismo prompt semántico) — cacheo por embedding de la query; ahorra costo y latencia. Y el punto que casi nadie cubre: **degradación cuando el modelo falla o el GPU se agota** — fallback a un modelo más barato, o a la lógica clásica sin IA, y el breaker para no esperar 3s por un modelo caído. El costo también se diseña: cacheo + batching + el modelo más pequeño que cumpla el SLO. La IA en el hot path es un problema de *latencia y degradación*, no de 'llamar a la API del modelo'."

**11. "¿Cómo haces para que tu equipo crezca? ¿Cómo revisas código?"**

**Orientación:** Mentoring y leverage (dimensión humana del senior). El senior habla de *hacer escalar a otros*, no de hacer más él mismo; y del code review como herramienta de consenso, no de caza.

**Respuesta de un senior:** "Mi trabajo como senior no es producir más código: es **multiplicar** al equipo. En code review, la regla es que el review entrena al autor: no escribo la solución, hago preguntas que la guían ('¿qué pasa si este servicio se cae?', '¿por qué eligió esta estructura?'), discuto *trade-offs*, no gustos (un estilo que no me gusta no es motivo de bloqueo), y apruebo rápido lo que está bien. Para el crecimiento: **delego con red de seguridad** — doy ownership de una feature completa a un junior con un contrato de alcance, y reviso los puntos de riesgo en vez de cada línea; explico el *por qué* de las decisiones, que es lo que convierte instrucción en criterio. Y hago **code reviews de decisiones**: los ADRs (Architecture Decision Records) se discuten antes de implementar, no después. El criterio de éxito: el equipo puede operar sin mí — no que no me necesiten, sino que el conocimiento no depende de una persona."

**12. "Tu métrica de negocio clave (conversión, éxito de pagos) se desploma un 20% sin alarma. ¿Qué haces?"**

**Orientación:** Observabilidad + operational judgment (M07). El senior: la métrica de negocio como SLI, primero descartar el falso positivo (deploy? data?), acotar, y encontrar el *por qué* detrás del número.

**Respuesta de un senior:** "Primero, **confirmo que no es un artefacto**: ¿hubo deploy? ¿cambio de datos? ¿cambio de tracking? ¿es conversión global o de un segmento? (un cambio en el SDK de tracking produce 'caídas' falsas). Acoto por *dimensiones*: por página, por país, por versión de app, por hora — el patrón del desplome me dice dónde mirar. Luego correlaciono con las otras señales: ¿el 20% de conversión bajó porque *el tráfico* bajó o porque *la conversión por visita* bajó? Si el tráfico bajó, es adquisición o indexación; si la conversión por visita bajó, es el funnel — un endpoint del checkout, una feature flag mal activada, un error en el pago. La lección operativa: una métrica de negocio **sin alerta es un incidente esperando ocurrir** — por eso las alertas deben vigilar SLIs de negocio (éxito de pago, conversión), no solo CPU. Y el arreglo final incluye siempre: ¿por qué no nos avisó la alarma? La detección tardía es parte del postmortem."

**13. "Diseña la evolución de un esquema de base de datos sin downtime."**

**Orientación:** Migraciones expand/contract, compatibilidad hacia adelante, y cuándo es inevitable la ventana de mantenimiento (M12). El senior habla de aditivo y de desacoplar deploy del cambio de esquema.

**Respuesta de un senior:** "La regla es **expand/contract**: nunca rompo el esquema en un paso. Primero **expand**: añado la columna nueva (nullable o con default) — el código nuevo la escribe; el código viejo sigue funcionando porque no la toca. Luego convivo: los dos códigos (viejo y nuevo) corren contra el esquema ampliado, con **compatibilidad hacia adelante** (la app vieja ignora lo desconocido). Después **migro los datos** en segundo plano con batches (backfill), nunca en un solo UPDATE gigante. Finalmente **contract**: una vez que el código viejo no existe, quito la columna o aplico el constraint. El truco con los constraints: añadir un `NOT NULL` con default es seguro; un `CHECK` o `UNIQUE` sobre millones de filas se hace con `CONCURRENTLY` (Postgres) para no bloquear. Si la migración es *destructiva* (renombrar/transformar), el rollback no basta — por eso se evita. Y por qué importa en producción: el deploy es continuo y no puedes asumir que la app se apaga; el esquema evoluciona *con* el código, no en un paso."

**14. "¿Qué haces cuando te asignan un requisito imposible o mal planteado?"**

**Orientación:** Juicio e influencia sin autoridad. El senior separa *problema* de *solución propuesta*, cuantifica el costo real y negocia el alcance.

**Respuesta de un senior:** "No digo 'no se puede' a secas: **separo el problema del camino propuesto**. Si me piden algo con un plazo imposible, lo primero es entender *qué objetivo de negocio* persigue y *cuál es la restricción real* — muchas veces el 'requisito' es una solución que alguien inventó y hay un camino más barato. Luego **cuantifico el trade-off en términos de negocio**: 'ese alcance en esa fecha cuesta X; con esto ya no se puede; o recortamos alcance, o movemos la fecha, o aceptamos el riesgo con un plan de mitigación'. Nunca digo que algo es imposible sin proponer alternativas (reducción de alcance, faseado, MVP). Y si es un requisito técnicamente mal planteado (una feature que romperá la consistencia), lo explico con consecuencias concretas —'esto va a perder dinero en doble-cobro'— no con principios abstractos. Mi criterio: aceptar el *problema*, negociar el *alcance*, y documentar la decisión. Decir sí a todo es deuda; decir no sin propuesta es inutilidad."

**15. "¿Cómo priorizas cuando hay tres incidentes a la vez y tú eres el on-call?"**

**Orientación:** Operational judgment bajo presión. El senior prioriza por **blast radius y RTO**, no por orden de llegada; y escala, no hace todo solo.

**Respuesta de un senior:** "Primero **escalo el que puede escalarse**: si hay tres incidentes, un on-call solo no es el estado correcto — el primer acto es *invocar la escalada* (secundario, y el resto del equipo si hace falta). Luego priorizo por **blast radius**: ¿cuál afecta a más usuarios o pierde más dinero *ahora*? (el checkout caído gana sobre el reporte de analytics, siempre). Después **estabilizo antes que investigar**: la prioridad es mitigar (rollback, feature flag, re-ruta), no la causa raíz — el root cause es para el postmortem. Y por cada incidente dejo un **único dueño** y una actualización pública cada X minutos; los tres se documentan, pero la atención se concentra en el de mayor impacto. El criterio que uso para 'mayor impacto': usuarios afectados × severidad del daño × velocidad de sangrado. Y cuando lo urgente pasa, cada incidente cierra con su postmortem — si solo apago fuegos y no arreglo las alarmas que no sonaron, el cuarto incidente es culpa mía."

**16. "¿Cuándo usarías una base de datos NoSQL en vez de SQL?"**

**Orientación:** Trade-off reasoning avanzado (M05). El senior responde por *access patterns* y *garantías*, con ejemplos concretos y los costos de cada uno.

**Respuesta de un senior:** "No elijo por 'NoSQL es más moderno': elijo por **access pattern y garantías**. SQL cuando hay relaciones, joins, ACID y consistencia fuerte — pedidos, pagos, inventario, cualquier cosa con transacciones entre entidades. NoSQL (clave-valor/documento/columna) cuando el acceso es **por clave** y necesito **escala horizontal nativa** y write-heavy: perfiles (`get by user_id`), sesiones, feed materializado, última ubicación GPS (DynamoDB/Cassandra), o documentos autónomos que no necesitan joins (Mongo). La regla que aplico: si el patrón de acceso es *lookup por clave* → NoSQL; si es *query relacional con joins* → SQL. Y el costo que pago al ir a NoSQL: consistencia eventual, sin transacciones multi-fila (o muy limitadas), y *el modelo se diseña a mano* (denormalización, secondary indexes que son otra tabla). El matiz senior: muchos sistemas usan ambos — un perfil en DynamoDB con el feed materializado, y los pedidos en Postgres. Elegir bien es saber *qué garantía necesita cada dato*."

**17. "¿Cómo garantizas que una feature nueva no rompa una que ya existe?"**

**Orientación:** Testing, feature flags, canary, observabilidad (M12/M13). El senior combina *mecanismos* (tests, CI, canary) con *medición* (comparar la nueva contra la vieja).

**Respuesta de un senior:** "Capas de protección en orden de costo. Primero: **tests** en CI — unitarios rápidos y los de integración que cubren los contratos; si la feature toca un acceso a datos, test que verifique el SQL real. Segundo: **feature flags** — despliego el código apagado y lo activo por segmento, así el deploy no es el evento de riesgo. Tercero: **canary** — dejo pasar un pequeño % del tráfico a la versión nueva y comparo métricas (latencia, errores, *y métricas de negocio*: conversión, éxito de pagos) contra la versión vieja antes de promocionar; el canary no es 'exponer y ver', es 'comparar contra una línea base'. Cuarto: **observabilidad por feature** — la feature nueva expone sus propias métricas para que una regresión se vea en el dashboard antes que en las quejas. Y el principio que lo sostiene todo: **compatibilidad hacia adelante** — la app nueva debe funcionar con datos de la vieja y viceversa. Si la feature es arriesgada, le añado un *shadow mode*: se ejecuta en paralelo con la vieja y se comparan los resultados sin afectar al usuario."

**18. "¿Cuál es tu opinión sobre las herramientas de IA en el desarrollo? ¿Cómo cambia tu trabajo?"**

**Orientación:** AI-aware (2026). El senior no es ni Luddite ni fan: la IA acelera la *escritura*, y su valor sube en la *definición, revisión y evaluación* (M13). Conecta con juicio humano.

**Respuesta de un senior:** "La IA (Copilot, Claude Code, Cursor) acelera la *escritura* de código, que es la parte barata y conocida; mi valor como senior está en las partes que la IA no hace: **definir el problema, proveer contexto, y evaluar si lo generado es correcto**. Los datos lo dicen: SWE-bench subió de ~13% a ~78-80% de issues resueltos en dos años, pero la tasa de PRs aceptados en producción real se estima en 35-50% — ese hueco entre benchmark y realidad es exactamente el territorio del senior: el código generado necesita *evaluación, testing y revisión*. En mi flujo: uso la IA para el contexto (AGENTS.md, specs), para velocidad en lo mecánico y para tests, pero **el código que entra a producción pasa por el mismo review y las mismas garantías que el escrito a mano** — el nivel de calidad no cambia porque la fuente sea un modelo. Y gestiono el riesgo: el código generado es confiable *a veces*, y todo lo que genera necesita pruebas. La IA no reemplaza el juicio; lo hace más valioso — alguien tiene que decidir qué se integra."

---

## Referencias

- **KORE1 — "Backend Engineer Interview Questions 2026 (by Level)".** Las 5 dimensiones del rubric y su peso por nivel; la calibración junior/mid/senior/staff; el patrón "then I'll break it".
- **ShadeCoder — "Debugging Round Interview Questions (2026)".** Los tres tipos de debugging round, el proceso reproduce→narrow→hypothesis→fix→verify, y los errores que eliminan candidatos.
- **MockIF — "Debugging Interview Questions".** Qué se puntúa en una ronda de debugging y los fallos comunes (editar sin entender, depurar en silencio, arreglar el síntoma).
- **Hunt, A. y Thomas, D. — *The Pragmatic Programmer*.** El capítulo de Debugging: *Fix the Problem, Not the Blame*, *Don't Panic*, *Failing Test Before Fixing Code*, *Don't Assume It — Prove It*, rubber ducking, y el debugging checklist.
- **CareerLift — "How to Prepare for a Backend Engineer Interview in 2026".** El loop backend (tech screen → coding → system design → database → behavioral) y el plan de preparación de 7 semanas.
- **Kleppmann, M. — *Designing Data-Intensive Applications*.** Profundidad de los fundamentos que un senior debe poder razonar (replicación, sharding, consistencia, outbox).
- **Richards, M. y Ford, N. — *Fundamentals of Software Architecture*, 2nd ed.** El juicio arquitectónico: trade-offs, *why over how*, decisiones con información incompleta.
- **KORE1 — "System Design Interview Questions for Senior Engineers 2026".** Las AI infrastructure questions como estándar 2026.
- Módulos 01–14 de esta guía, cuyos contenidos son la materia prima de todas las respuestas de este banco.
