# Módulo 16 — Roadmap de estudio + proyecto final

> **Nivel:** Senior. Este módulo es el _puente de salida_: toma todo lo aprendido en los módulos 01–15 y lo organiza en un **roadmap de 10 semanas** con un **proyecto final integrador** (un subsistema _order & fulfillment_ end-to-end sobre AWS) que fuerzas a aplicar cada módulo — desde arquitectura hasta CI/CD, pasando por AI-assisted development y observabilidad. No es una checklist de teoría: es **el portfolio** que demuestra que puedes diseñar, construir, desplegar y operar un sistema real como lo haría un ingeniero senior en Amazon, Stripe o Mercado Libre.

> **Conexiones:** este módulo **consume** todo lo anterior — [Módulo 14](14-System-Design-Interview.md) para el framework de diseño, [Módulo 15](15-Preguntas-y-Ejercicios.md) + [Apéndice A](appends/15-Preguntas-y-Ejercicios-Apendice-A.md) para los ejercicios de cada paso, y los [Apéndices A](appends/01-Arquitectura-de-Software-Moderna-Apendice-A.md) de los módulos para las implementaciones. El [Anexo A](appends/16-Roadmap-y-Proyecto-Final-Apendice-A.md) contiene los **laboratorios integradores** (multi-módulo) y el **laboratorio de ingeniería agencial**.

---

## Introducción

### Por qué este proyecto (y no otro)

La diferencia entre un ingeniero que _conoce_ arquitectura y uno que _ejerce_ arquitectura es un **artefacto comprobado**: algo que diseñaste, construiste, desplegaste, rompiste y arreglaste en producción simulada. Este capstone no es un "hola mundo": es un subsistema el cual:

- **tiene concurrencia real** (miles de pedidos simultáneos, hot keys en stock),
- **necesita consistencia correcta** (pago doble-cobro vs. perder un pedido) y consistencia eventual donde conviene (feed de tracking),
- **tiene observabilidad operativa** (alertas, SLOs, un incidente simulado de race condition),
- **se integra a una pipeline de CI/CD** con canary y feature flags,
- **se sirve con IA** (revisión de diffs, generación de tests, evaluación de riesgos),
- y **se entrevista como black-box** (presentas el diseño y un entrevistador "te lo rompe").

El "order & fulfillment" cumple con todo: toca stock, pagos, estado de pedido, tracking, notificaciones y una cola de trabajo. Es el _Twitter/X_ del módulo 14 en miniature: todos los patrones, una fracción del alcance.

### Cómo usar este roadmap

1. **Seguía el orden.** Cada semana depende de la anterior (no saltes la semana 2 de CI/CD si no hiciste la semana 1 de dominio).
2. **No copies; escribe de nuevo.** El objetivo es internalizar los patrones, no tener un repo de ejemplo. Si te mueves de un módulo a otro sin escribir, no aprendiste.
3. **Rompe el capstone, arregla el capstone.** La semana 8 simula un incidente real; la 9 lo revisas como postmortem.
4. **Graba tu presentación de system design (semana 10).** En voz alta, sin apuntes, como si fuera una entrevista. Si no puedes explicar el porqué de una decisión, no la diste.

---

# Roadmap de estudio de 10 semanas

> Cada semana bloquea 6–10 h. El capstone se construye incrementalmente: **no diseñes todo de golpe** (error senior junior), sino que dejes que el diseño evolucione feature a feature, como lo harías en un equipo real.

| Semana | Enfoque                    | Módulos de referencia | Hitos del capstone                                                           |
| ------ | -------------------------- | --------------------- | ---------------------------------------------------------------------------- |
| 0      | Setup y mental model       | 01, 13, 16            | Repo con AGENTS.md, spec markdown, CI de guardia; roadmap de estudio firmado |
| 1      | Domina el dominio          | 01, 02                | Modelo del dominio: `Order`, `Stock`, `Payment`; Bounded Context `order`     |
| 2      | Paga sin doble cobro       | 05, 09, 14            | Endpoint `POST /orders` con idempotency key + double-entry ledger de stock   |
| 3      | Procesa sin perder nada    | 03, 10, 12            | Pipeline de events `order.created` → fulfillment vía SQS + DLQ               |
| 4      | Sirve y observa            | 04, 07, 14            | Lambda/Fargate + OpenTelemetry + RED/SLOs + alertas de negocio               |
| 5      | Escala y resiste           | 10, 05, 14            | Rate limiter, circuit breaker, cache-aside, sharding por tenant              |
| 6      | Asegura y libera           | 08, 12, 11            | Auth OIDC, IAM least-privilege, secretos, rollback sin redeploy              |
| 7      | Despliega sin susto        | 12, 01, 14            | CI/CD canary + feature flag + migration expand/contract                      |
| 8      | **Incidente simulado**     | 15 (debugging)        | Reproducir, acotar, corregir, postmortem                                     |
| 9      | Refuerza contra el fracaso | 10, 15                | Tests de contrato, chaos, hardening del hot path                             |
| 10     | **Presenta como senior**   | 14, 15                | Presentación de system design + defensa bajo "te lo rompo"                   |

---

## Semana 0 — Setup y mental model

**Objetivo:** un repo profesional con _guardias de calidad_ desde el día uno. El setup es donde un senior pone las bases para que el crecimiento no dependa de él.

**Qué entregas:**

- `AGENTS.md` (ver [M13](#conexiones-y-plantilla-de-agents-md)) con: estructura del repo, stack (TypeScript + AWS CDK vía CDKTF o Terraform, Python runner de tests), convenciones de branch/PR, comandos de setup y _el contrato de calidad_ (qué pasa si falla un test, un lint, un escaneo). Usa el patrón del [M13 Apéndice A](appends/13-AI-Assisted-Development-Apendice-A.md).
- `spec/` con un **spec markdown** del capstone: _who/what/why/how much_ (alcance, FRs, NFRs, escala esperada, out of scope). Esto es **context engineering para humanos** — evita que el proyecto gane forma propia.
- CI mínima: `lint + typecheck + test` en cada PR, con quality gates (coverage mínima, SAST). Si no tienes esto en la semana 0, el resto del roadmap se hunde en deuda.
- `ROADMAP.md` con tu hoja de ruta firmada (qué harás, cuándo, y qué indica éxito). Un senior documenta el plan y sus supuestos.

> **Señal senior en setup:** no uses el setup como excusa para no empezar. Un repo sin CI es un repo en el que nada confía.

## Semana 1 — Domina el dominio (no el framework)

**Objetivo:** que tu código refleje el lenguaje del negocio, no el de la base de datos. El 90% de los bugs vienen de un mal modelo de dominio.

**Módulos clave:** [M01](01-Arquitectura-de-Software-Moderna.md) (arquitectura centrada en el dominio, ADRs), [M02](02-Microservicios-y-DDD.md) (Bounded Contexts, Aggregates, descomposición).

**Hitos del capstone — `order` context:**

- **Bounded Context único** que contiene todo lo del pedido (no un microservicio por tabla). El límite correcto de un servicio viene de _una razón de cambio_, no de "una tabla = un servicio".
- **Aggregate raíz:** `Order`. Sus invariantes: el stock no puede quedar negativo; un pedido no cambia de estado a un estado inconsistente (`cancelled` no vuelve a `paid`).
- **Value Objects:** `OrderId`, `Money`, `Address` (inmutables, validados en el constructor).
- **Domain Events:** `OrderCreated`, `OrderConfirmed`, `OrderCancelled` — el _qué pasó_, no el _qué hacer_.
- **ADR:** "por qué un solo Bounded Context y no microservicios todavía" → grábalo como `docs/adr/001-context-boundaries.md`. Un junior cambia de idea; un senior registra su decisión.

> **Ejercicio del apéndice:** [3.1](#apéndice-a) (diseño de estructuras bajo presión) y la checklist de [M15 Apéndice A](appends/15-Preguntas-y-Ejercicios-Apendice-A.md) ("¿qué hace falta para que un dominio sea testeable?").

**Cómo usar la IA (semana 1):** usa un agente para que te ayude a escribir los Value Objects con validación, pero **tú** defines el Aggregate y revisas los diff de la IA (ver [M13](#construyendo-con-ia)). No delegues el _diseño del Aggregate_.

## Semana 2 — Paga sin doble cobro, sin perder pedidos

**Objetivo:** que la escritura de un pedido y la reserva de stock sean **atómicas o no happen**, y que el cliente nunca vea un cobro doble por un retry. La peor feature del mundo es "a veces cobra dos veces".

**Módulos clave:** [M05](05-Bases-de-datos-distribuidas.md) (isolation, conditional writes), [M09](09-Diseño-de-APIs.md) (idempotency key), [M14](14-System-Design-Interview.md) (double-entry ledger).

**Hitos del capstone — `POST /orders`:**

- **Idempotency key:** `POST /orders` con `Idempotency-Key`. Usa la técnica de la [sección 1.2 del apéndice](appends/15-Preguntas-y-Ejercicios-Apendice-A.md): PK única = la key; insert atómico; si existe, devolver resultado previo o `409` (procesando). **Nunca** procesar de nuevo.
- **Stock como double-entry:** cada reserva de stock es un asiento `inventory_held +1` / `order_reserved +1` (partida doble). Si la transacción no cuadra, algo está mal — y lo sabes al instante.
- **Isolation adecuado:** la validación del stock (`stock >= qty`) y el decremento en la misma operación atómica (`UPDATE ... WHERE stock >= qty`), evitando el TOCTOU de la [sección 2.1](appends/15-Preguntas-y-Ejercicios-Apendice-A.md).
- **Test:** el caso de prueba es _dos requests concurrentes del último artículo_ → exactamente uno pasa. Si tu test no reproduce la condición de carrera, no está completo.

> **Pregunta de senior que debes poder responder:** "¿por qué no hago el stock en memoria con un mutex?" → porque no es atómico con la DB, no sobrevive a un restart y no distingue retries de procesos distintos.

## Semana 3 — Procesa sin perder nada (eventos + colas)

**Objetivo:** el pedido se crea y, **sin bloquear al cliente**, se dispara el pipeline de fulfillment (reservar stock, cobrar, notificar). Todo es _at-least-once_, el consumidor es idempotente.

**Módulos clave:** [M03](03-Event-Driven-Architecture.md) (outbox, saga, semántica de entrega), [M10](10-Escalabilidad-y-Resiliencia.md) (retry, circuit breaker), [M12](12-CI-CD-Moderno.md) (deploy sin downtime).

**Hitos del capstone:**

- **Outbox:** el `INSERT` del pedido y el evento `OrderCreated` en la misma transacción (ver [glosario: Outbox](#outbox-patrón--transactional-outbox)). El relay (CDC o poller) publica a SQS; el consumidor de fulfillment **deduplica por `event_id`** (at-least-once + idempotencia = efectivamente-once).
- **DLQ operativa:** lo que falle de forma permanente va a la DLQ y **dispara una alerta**. El replay lo clasifica (transitorio → re-encola; schema → archiva). Usa el [ejercicio 1.9](appends/15-Preguntas-y-Ejercicios-Apendice-A.md) como guía.
- **Saga:** el checkout es una saga de 3 pasos (reservar stock → cobrar → confirmar); si el paso 3 falla, la compensación libera el stock y reembolsa. **Orquestación** (un state machine en Step Functions te da visibilidad de pila).

> **Pregunta de senior:** "¿por qué no publico el evento directamente a SQS después del commit de la DB?" → porque un crasheo entre ambos deja el pedido sin evento (dual write). El outbox lo resuelve en la misma transacción.

## Semana 4 — Sirve y observa como lo haría un on-call

**Objetivo:** el endpoint responde rápido y, si no lo hace, **alguien se entera antes que el usuario**. La observabilidad no es opcional: es el contrato con el negocio.

**Módulos clave:** [M04](04-AWS-Serverless.md) (Lambda/Fargate, cost reasoning), [M07](07-Observabilidad.md) (otel, RED/SLO, alertas), [M14](14-System-Design-Interview.md) (latencia del hot path).

**Hitos del capstone:**

- **Servicio express:** API Gateway → Lambda (o Fargate) para `GET /orders/{id}`, con el order materializado en una tabla de lectura para O(1).
- **OpenTelemetry:** el `trace_id` viaja del cliente al consumidor del fulfillment; cada span lleva atributos (`order_id`, `tenant_id`).
- **RED por servicio + SLOs de negocio:** `order.created→confirmed` en < 5 s p95; `payment success rate` > 99.5%. La alerta es sobre el **error budget agotado**, no sobre CPU al 100%.
- **Métricas de negocio como SLI:** el dashboard muestra `conversion rate` y `payment success` con un `429`/`500` breakdown — porque una caída del 20% en conversión sin alarma (ver [M15 pregunta 12](15-Preguntas-y-Ejercicios.md)) es el incidente que un senior evita.

> **Ejercicio del apéndice:** ve el [laboratorio de observabilidad](#apéndice-a) para configurar promql de histogramas y un dashboard RED en 60 minutos.

## Semana 5 — Escala y resiste (el hot path no se rompe)

**Objetivo:** el subsistema aguanta picos (Black Friday simulado: 10× el tráfico normal) sin caer, y un `stock_hotkey` no derriba una sola partición.

**Módulos clave:** [M10](10-Escalabilidad-y-Resiliencia.md) (shard, rate limiter, breaker, stampede), [M05](05-Bases-de-datos-distribuidas.md) (partition key), [M14](14-System-Design-Interview.md) (hot key).

**Hitos del capstone:**

- **Rate limiter distribuido:** límite global por tenant con token bucket en Redis (ver [sección 3.1](appends/15-Preguntas-y-Ejercicios-Apendice-A.md)); `429 + Retry-After`.
- **Circuit breaker + bulkhead:** el proveedor de pagos caído no satura tu pool de hilos ([sección 1.8](appends/15-Preguntas-y-Ejercicios-Apendice-A.md); el [laboratorio 3.2](#apéndice-a) cubre el thread starvation).
- **Cache-aside con stampede protection:** `GET /orders/{id}` lee del cache first; en miss, el cache-aside con lock + TTL con jitter evita la avalancha (ver [sección 1.4](appends/15-Preguntas-y-Ejercicios-Apendice-A.md)).
- **Sharding por tenant:** el `partition key` de DynamoDB incluye el `tenant_id` → hot keys de un tenant no afectan a otros. El stock del producto estrella usa un "counter table" con write sharding para no saturar una partición.

> **Señal senior:** nombrar el hot key y su mitigación _antes_ que te lo pregunten. Si tu diseño no menciona hot keys, no lo has pensado.

## Semana 6 — Asegura y libera el acceso (no el dato)

**Objetivo:** tu API es pública; no confíes en la red perimetral. Cada request porta su identidad, y cada credencial es mínima y efímera.

**Módulos clave:** [M08](08-Seguridad.md) (OIDC, JWT, IAM least-privilege), [M12](12-CI-CD-Moderno.md) (OIDC keyless a AWS), [M01](01-Arquitectura-de-Software-Moderna.md) (separación de capas).

**Hitos del capstone:**

- **Auth OIDC con PKCE:** el cliente (SPA/móvil) hace PKCE contra Cognito; la Lambda recibe un JWT y **valida firma + `exp` + `aud`** localmente (ver [M08 Apéndice A](appends/08-Seguridad-Apendice-A.md)).
- **IAM least-privilege:** la función Lambda asume un rol con _exactamente_ los permisos que necesita (un `PutItem` a una tabla, nada más); usando OIDC keyless, los runners de CI no guardan access keys.
- **Secretos:** el `Idempotency-Key` y los secrets van a Secrets Manager / Parameter Store, **nunca** en el repo. El webhook signing secret también.
- **Security gates en CI:** SAST + SCA como quality gates (ver [M12 Apéndice A](appends/12-CI-CD-Moderno-Apendice-A.md), quality gates). Un security gate que no falla el build es decoración.

## Semana 7 — Despliega sin susto (canary + feature flag + migraciones)

**Objetivo:** tu cambio no rompe lo que existe. El deploy no es el evento de riesgo; el _release_ (feature visible) lo es, y se controla con flag.

**Módulos clave:** [M12](12-CI-CD-Moderno.md) (canary, feature flag, blue/green), [M05](05-Bases-de-datos-distribuidas.md) (migraciones), [M01](01-Arquitectura-de-Software-Moderna.md) (fitness functions, contract testing).

**Hitos del capstone:**

- **Canary con telemetría:** el 5% de usuarios ve la nueva versión; la promoción se valida con métricas (latencia, error rate) antes del 100%. Usa la plantilla del [M12 Apéndice A](appends/12-CI-CD-Moderno-Apendice-A.md).
- **Feature flag con rollout determinista:** una feature nueva se mergea apagada y se activa por porcentaje de usuario (hash `(flag, user_id)`) — ver [sección 1.8](appends/15-Preguntas-y-Ejercicios-Apendice-A.md).
- **Migración expand/contract:** añadir la columna `customer_country` → expand (nullable + código nuevo la escribe) → backfill en background → contract. Usa el [ejercicio 1.10](appends/15-Preguntas-y-Ejercicios-Apendice-A.md) como checklist. `ALTER TABLE ... ADD COLUMN` no bloqueante, y `CONCURRENTLY` para constraints.
- **Contract tests:** el `API contract` entre order y fulfillment se valida en CI (Pact/Schemathesis) — si rompes el contrato, el build lo sabe.

> **Pregunta de senior que debes poder defender:** "¿por qué no haces blue/green con la base de datos compartida?" → porque el rollback de datos es el peor escenario; prefiero migraciones aditivas que permitan rollback de app sin tocar datos.

## Semana 8 — Incidente simulado (la entrevista de debugging)

**Objetivo:** romper el capstone con un bug real y resolverlo con el proceso (no con adivinanzas). Simula que un usuario reporta: "a veces me cobra dos veces y el pedido no se crea".

**Módulos clave:** [M15 Apéndice A](appends/15-Preguntas-y-Ejercicios-Apendice-A.md) (proceso de debugging), [M15](15-Preguntas-y-Ejercicios.md) (debugging under ambiguity).

**Escenario del incidente:** un bug de concurrencia en el _payment retry_ — el cliente reintenta por timeout, el proveedor procesa de todos modos, y el idempotency key del cliente no coincide con el del evento del webhook → **doble cobro**.

**Tu proceso (a seguir exactamente):**

1. **Reproducir:** test de concurrencia que simule "timeout del cliente + 2 webhooks" → reproduce el doble cobro. _Si no puedes reproducir, no sabes si queda arreglado._
2. **Acotar:** el idempotency key del _cliente_ (`Idempotency-Key` del `POST /orders`) difiere del evento del webhook del _proveedor_ → dos paths con keys distintas.
3. **Hipótesis falsable:** "el dedupe falla porque el webhook no lleva la misma key que el cliente".
4. **Corregir causa raíz:** el `event_id` del webhook es la fuente de dedupe para el consumidor; el cobro duplicado se evita con el idempotency key del proveedor (`charge_id`), no el del cliente. Aseguras que el webhook incluye `charge_id` y el consumidor dedupe por él.
5. **Verificar:** el test de concurrencia pasa; añades un test de regresión.
6. **Postmortem:** ¿por qué no se detectó? → no habías testado el path de webhook retry. Lo añades al contract test de la semana 7.

> **Regla de esta semana:** no mires el código a ciegas. Verbaliza reproduce→acota→hipótesis→verifica (el [diagrama de proceso de debugging](appends/15-Preguntas-y-Ejercicios-Apendice-A.md)). Un senior no "arregla el síntoma"; arregla la _condición_ que permite el doble cobro.

## Semana 9 — Refuerza contra el fracaso (tests + chaos)

**Objetivo:** tu sistema no pasa a producción sin tests automatizados y una historia de _lo que rompe y cómo lo detectas_.

**Módulos clave:** [M01 Apéndice A](appends/01-Arquitectura-de-Software-Moderna-Apendice-A.md) (testing piramidal), [M10](10-Escalabilidad-y-Resiliencia.md) (chaos), [M13](13-AI-Assisted-Development.md) (tests asistidos).

**Hitos del capstone:**

- **Tests de contrato** entre servicios (order ↔ fulfillment) → el contrato vive en el repo y se valida en CI.
- **Testing piramidal con calidad:** unitarios rápidos que cubren los Aggregates; integration tests que ejecutan el _real_ SQL (testcontainers) y la _real_ cola (LocalStack); **pocos e2e, todos observables**.
- **Property-based tests** para el rate limiter y el idempotency store (con `fast-check`): "para cualquier sequence de N requests concurrentes, el número de éxitos ≤ capacidad".
- **Chaos engineering:** mata una instancia de Lambda/Fargate en el poller del outbox; la DLQ crece y la alerta de DLQ se dispara; el postmortem de la semana 8 pasa a runbook.
- **Hardening del hot path:** el `GET /orders/{id}` tiene cache-aside + circuito abierto ante el DB; el timeout al proveedor de pago es 2 s (no 30 s). Usa [las pruebas del apéndice](appends/15-Preguntas-y-Ejercicios-Apendice-A.md) como checklist.

> **`fast-check` no es opcional:** un senior prueba _propiedades_, no solo casos.

## Semana 10 — Presenta como senior (system design defensiva)

**Objetivo:** presentar el capstone en voz alta, como si fuera una entrevista, y defender cada decisión bajo el patrón _"OK, eso está bien… pero ahora te lo rompo"_ (ver [M15](15-Preguntas-y-Ejercicios.md) sobre el rubric 2026).

**Módulo clave:** [M14](14-System-Design-Interview.md) (framework de 4 pasos, cost reasoning, trade-offs).

**Tu presentación** (graba un video de 5 minutos o hazlo en voz alta frente a un espejo):

1. **Scope & estimación (back-of-the-envelope):** "25M pedidos/día, 2.5K QPS pico 10×; hot path < 100 ms p95." Declara los supuestos (1 KB por pedido, ratio read:write 100:1).
2. **High-level (4 cajas):** Gateway → Order service → (DB + outbox/queue) → Fulfillment worker → Payment provider. Nombra el hot path de cada request.
3. **Componentes críticos:** el idempotency key (evita doble cobro), el outbox (garantiza el evento), el saga de compensación (recovery), el rate limiter shardado.
4. **Deep dive & stress:** el hot key del producto estrella; el _what if_ de "el proveedor de pagos cae" (breaker + DLQ + fallback); "el webhook llega duplicado".
5. **Trade-offs nombrados:** "DynamoDB en vez de Postgres para el pedido: escala nativa y write heavy, a cambio de consistencia eventual y joins por aplicación. No para el ledger de pagos — ese sigue en Postgres con ACID."
6. **Cost reasoning (rubric 2026):** "el costo dominante es el ancho de banda y los invocations de Lambda; mitigo con cache-aside y el 95% de los reads desde Redis."

**Cómo se rompe un capstone de junior (y cómo tú evitas los errores):**

| Pregunta del entrevistador ("te lo rompo")       | Respuesta junior      | Respuesta de senior (requiere profundidad operativa)                                                                |
| ------------------------------------------------ | --------------------- | ------------------------------------------------------------------------------------------------------------------- |
| "¿y si el proveedor de pagos cae por 5 minutos?" | "reintentamos"        | rate limiter pausa + DLQ acumula + fallback a reserva de stock con compensación posterior + reconciliación post-cae |
| "¿cómo evitas el hot key del producto estrella?" | "cache"               | counter table + write sharding + cache-aside + jitter en TTL                                                        |
| "¿qué pasa si el outbox se llena?"               | "se cuelga todo"      | alerta de backlog de outbox; circuito abierto + drop controlado + deadletter interno                                |
| "¿el webhook del proveedor no llega?"            | "el cliente se queja" | replay de webhooks con idempotency + reconciliación por ventana + alerta de "webhook sin ack a los 5 min"           |
| "¿cómo sabes que no hay doble cobro?"            | "no pasa"             | métrica de `charge_id` única + alerta de duplicidad + audit log en el ledger                                        |

> **La prueba final:** si no puedes explicar _por qué elegiste X en vez de Y_ en 20 segundos, no lo diste tú — y un entrevistador lo notará. Un senior defiende su elección con _costo + requisito + trade-off_, no con "porque se usa así".

---

## Construyendo con IA (capa sobre todo el roadmap)

> La IA no diseña por ti, pero sí **acelera el escribir** si tú controlas el qué, el cómo y el _por qué_. Aplica esto en cada semana, no solo en la 13.

**Plantilla de AGENTS.md de referencia (usa el del [M13 Apéndice A](appends/13-AI-Assisted-Development-Apendice-A.md) como base):**

```markdown
# AGENTS.md — Order & Fulfillment subsystem

## Stack

- Runtime: Node.js 20 + TypeScript + AWS CDK
- Tests: Vitest (unit), Testcontainers (integration), fast-check (property)

## Quality contract

- Lint, typecheck, test must pass (coverage ≥ 80%)
- SAST/DAST gates on CI

## Conventions

- Aggregates in src/domain; never import infra from domain
- Every public handler gets an idempotency key or is naturally idempotent
- No AI-generated code in infra/security without human review

## Commands

- `pnpm install && pnpm test` (local)
- `pnpm deploy:dev` (CDK synth + diff)
```

**Tu flujo con agentes, semana a semana:**

- **Spec first:** el spec markdown (semana 0) y los ADRs (semana 1) son tu "prompt de verdad" — pasas specs, no tareas. Un agente que no entiende el _por qué_ escribe código de mala calidad.
- **Velocidad en lo mecánico:** usa un agente para los Value Objects, los tests de regresión, los handlers de error (ver [M13 Apéndice A](appends/13-AI-Assisted-Development-Apendice-A.md)). Pides _patrones_, no resultados.
- **No para la decisión:** la elección del Aggregate, la partición de DynamoDB, el diseño del outbox — eso **tú** lo decides. La IA ejecuta, tú validas: _"el código generado necesita pruebas, review y el mismo contrato de calidad que el escrito a mano"_ ([M13](13-AI-Assisted-Development.md)).
- **Revisión de diffs con AI-aware:** usa la IA para revisar _intentos de bypass_ (ej: un handler que salta el idempotency check por "optimización"). El agente no ve el _trade-off_; tú sí.
- **Cierre del bucle (semana 10):** el postmortem incluye siempre: _"¿la IA generó algo que no validé?"_ → si sí, se añade al quality gate.

> **El matiz del senior en la era IA:** la IA multiplicó tu productividad para _escribir_; tu valor subió para _definir, ejecutar y validar_. No confíes en diffs que no entiendes.

---

## Checklist final (¿qué debes poder mostrar al finalizar?)

- [ ] Repo público con `AGENTS.md`, spec markdown, ADRs y `ROADMAP.md`.
- [ ] Endpoint `POST /orders` con **idempotency key + double-entry ledger de stock + TOCTOU-free**.
- [ ] Pipeline `order.created → fulfillment` vía **outbox + SQS + DLQ** con consumidor idempotente y **saga con compensación**.
- [ ] Servicio de lectura + **OpenTelemetry completo** (trace_id de end a end) + **SLOs/alertas de negocio** (no solo CPU).
- [ ] **Rate limiter shardado, circuit breaker + bulkhead, cache-aside con stampede protection.**
- [ ] CI/CD con **canary + feature flag** + migración **expand/contract**.
- [ ] **Auth OIDC + IAM least-privilege + secretos fuera del repo + SAST.**
- [ ] Tests piramidales + **property tests** + **contract tests**.
- [ ] **Incidente simulado reproducido y arreglado** con proceso escrito (failing test first).
- [ ] **Chaos experiment** (mata un worker) con runbook.
- [ ] **Video de 5 minutos presentando el system design** con 5 trade-offs defensadas y los _nombres_ de los puntos de estrés.
- [ ] **Postmortem del capstone** en `docs/postmortems/` (qué aprendiste, qué harías distinto).

> **Criterio de éxito no negociable:** un ingeniero que nunca ha construido nada (aunque sepa todo el libro) no es senior. Un ingeniero que construyó esto, lo rompió y lo defendió, sí lo es — incluso si cometió errores que luego corrigió. El portafolio es la prueba.

---

## Referencias

- **Hunt, A. y Thomas, D. — _The Pragmatic Programmer_, cap. Debugging.** El proceso de debugging del M15 (Tip 31) se aplica al incidente simulado de la semana 8.
- **Kleppmann, M. — _Designing Data-Intensive Applications_.** Transacciones, isolation, particionamiento y consistencia para las semanas 2 y 5.
- **Richards, M. y Ford, N. — _Fundamentals of Software Architecture_, 2nd ed.** El juicio de trade-offs de la semana 10.
- **KORE1 — "Backend Engineer Interview Questions 2026 (by Level)".** El rubric de las 5 dimensiones y la calibración senior que estructuran la semana 10.
- **AWS Well-Architected Framework (2026) — Reliability, Performance Efficiency, Cost Optimization.** Pilares aplicados semanas 4, 5 y 7.
- **Stripe / GitHub / AWS — documentación de idempotency keys, webhooks y SQS/DLQ.** Patrones de producción que el capstone replica.
- Módulos 01–15 de esta guía y sus apéndices, cuya implementaciones (ver Anexo A) son la base de cada semana.

---

> _Más profundidad y laboratorios integradores + ingeniería agencial en el [**Apéndice A**](appends/16-Roadmap-y-Proyecto-Final-Apendice-A.md)._
