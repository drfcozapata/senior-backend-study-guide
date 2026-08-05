# Apéndice A — Módulo 09: Implementaciones de referencia

> Este apéndice complementa [09-Diseño-de-APIs.md](09-Diseño-de-APIs.md) con código concreto y adaptable: middleware de idempotencia con Redis, cursor pagination, errores RFC 9457, firma de webhooks y un spec OpenAPI de ejemplo.

---

## 1. Middleware de idempotencia con Redis (Node.js / Express)

Replica el patrón de Stripe: lock `SET NX`, replay, y **no** cachear requests de validación fallida.

```js
// idempotency.js
const crypto = require("crypto");

// Inserta la respuesta cacheada solo después de que la operación inició
async function idempotency(redis, { keyHeader = "Idempotency-Key" } = {}) {
  return (req, res, next) => {
    if (req.method === "POST") {
      const key = req.headers[keyHeader];
      if (!key) {
        return res.status(422).json({
          type: "https://api.example.com/errors/missing-key",
          title: "Idempotency-Key required", status: 422,
          code: "idempotency_key_required",
        });
      }
      req.idempotencyKey = key;
      req.idempotencyStore = {
        async getResult(body) {
          const cached = await redis.hgetall(`idem:${key}`);
          if (!cached || cached.status !== "done") return null;
          // Si el body difiere del original → 409 (ambigüedad de lógica cliente)
          if (cached.bodyHash && cached.bodyHash !== hash(body)) {
            return { conflict: true };
          }
          return { replay: cached };
        },
        async lock() {
          // SET NX EX 60: solo uno gana; los demás esperan/timeout
          return await redis.set(`idem:lock:${key}`, "processing", "NX", "EX", 60);
        },
        async complete(body, status, payload) {
          await redis.hset(`idem:${key}`, {
            status: "done", bodyHash: hash(body),
            resStatus: status, payload: JSON.stringify(payload),
          });
          await redis.expire(`idem:${key}`, 86400); // 24h, como Stripe
          await redis.del(`idem:lock:${key}`);
        },
      };
    }
    next();
  };
}
const hash = (obj) => crypto.createHash("sha256").update(JSON.stringify(obj)).digest("hex");
```

En el handler, `GET` el resultado, y si el key existe y coincide → replay; si no, `lock` y procede (con el constraint único en BD como respaldo):

```js
app.post("/v1/charges", async (req, res, next) => {
  const e = req.idempotencyStore;
  const prior = await e.getResult(req.body);
  if (prior) return prior.conflict ? res.status(409).end() : res.status(Number(prior.replay.resStatus)).json(JSON.parse(prior.replay.payload));
  if (!(await e.lock())) return req.idempotencyWait(res); // o 409 "processing"
  try {
    const result = await processCharge(req);          // validación + ejecución
    await e.complete(req.body, 201, result.body);      // solo cachear SI validó/ejecutó
    res.status(201).json(result.body);
  } catch (err) {
    if (isValidationError(err)) return res.status(400).json(problem(err)); // NO se cachea
    next(err); // 5xx tampoco se cachea para permitir reiniento correcto
  }
});
```

**Respaldo en BD** (seguridad si Redis cae): columna `idempotency_key` con `UNIQUE(account_id, idempotency_key)`; al insertar, `ON CONFLICT DO NOTHING` y recoger la fila existente:

```sql
INSERT INTO operations (account_id, idempotency_key, status, result)
VALUES ($1, $2, 'done', $3)
ON CONFLICT (account_id, idempotency_key) DO UPDATE SET result = operations.result
RETURNING *;
```

---

## 2. Cursor-based pagination (SQL)

Clave por un identificador monolótico. `LIMIT limit+1` para detectar `has_more` sin contar todo.

```sql
-- Cursor = base64(JSON/id) opaco
SELECT id, name, email
FROM users
WHERE id > :cursor           -- clave estable (índice PK)
ORDER BY id ASC
LIMIT :limit + 1;            -- fila extra para saber si hay más
```

```js
function toCursor(id) { return Buffer.from(`v1_${id}`).toString("base64url"); }
function fromCursor(cursor) { return Number(Buffer.from(cursor, "base64url").toString().replace("v1_", "")); }
```

Decisión del response:

```json
{
  "data": [ ... ],                       // se descarta la fila extra
  "pagination": {
    "next_cursor": "djFfODQ=",
    "has_more": true,
    "limit": 20
  }
}
```

Si el orden es por fecha, usa `(created_at, id)` como clave compuesta y un índice `(created_at, id)` — así el cursor estable incluye el desempate.

---

## 3. Errores RFC 9457 (Problem Details)

Handler central de errores con `application/problem+json` y discriminador accionable.

```js
function problem({ code, title, status, detail, instance }) {
  const problemDoc = {
    type: `https://api.example.com/errors/${code}`,   // URI estable
    title, status, code, instance,
  };
  if (detail) problemDoc.detail = detail;
  return problemDoc;
}

app.use((err, req, res, next) => {
  const instance = req.header("X-Request-Id") || `req-${crypto.randomUUID()}`;
  if (err.type === "validation") {
    return res.status(400).set("Content-Type", "application/problem+json")
      .json(problem({ code: "invalid_request", title: "Invalid request", status: 400, detail: err.message, instance }));
  }
  // 5xx: no exponer internals; dar Retry-After si es reintentable
  console.error("internal", { instance, err });       // log completo (Módulo 07)
  res.status(err.status || 500).set("Content-Type", "application/problem+json")
    .set("Retry-After", 5)
    .json(problem({ code: "internal_error", title: "Internal error", status: err.status || 500, instance }));
});
```

Ejemplo de respuesta `429` con headers de rate limit:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1712757600
Content-Type: application/problem+json

{ "type": ".../errors/rate_limit_exceeded", "title": "Rate limit exceeded",
  "status": 429, "code": "rate_limit_exceeded",
  "detail": "100 requests per minute limit reached", "instance": "req-9f21" }
```

---

## 4. Versionado por aislamiento + traducción

Dos lonas: el core solo habla la versión más reciente; un módulo transforma v1→v2.

```js
// version-transform.js — se ejecuta en el API Gateway / edger
const TRANSFORMS = {
  "2017-05-25": (resp, req) => ({ // convierte v1 (legacy) → core (v2)
    ...resp,
    total: resp.total_cents ? { amount: resp.total_cents, currency: "usd" } : resp.total,
  }),
};

function versionFor(req) {
  return req.get("Accept")?.match(/.*version=([\d-]+)/)?.[1]
    || req.get("X-API-Version")
    || "2017-05-25";                                   // default fijado por cuenta/cliente
}

// El core NO tiene ifs de versión:
function handlePayment(req) { return coreService.process(req); }

function expose(req, handler) {
  const v = versionFor(req);
  const result = handler(req);            // core devuelve forma v2 (más nueva)
  const transform = TRANSFORMS[v];
  return transform ? transform(result, req) : result;  // traduce v1 → v2
}
```

En respuestas, headers de deprecación machine-readable:

```
X-API-Version: 2017-05-25
Deprecated: true
Sunset: Sat, 19 Dec 2026 00:00:00 GMT
```

---

## 5. Firma de webhooks (para el integrador)

Del lado servidor: firma `HMAC-SHA256(secret, rawBody)` + timestamp. Del lado cliente: verificación + anti-replay + idempotencia por `event_id`.

```js
// Servidor: expone el payload firmado
function signWebhook(rawBody, secret) {
  const timestamp = Math.floor(Date.now() / 1000);
  const signature = crypto
    .createHmac("sha256", secret)
    .update(`${timestamp}.${rawBody}`, "utf8")
    .digest("hex");
  return { payload: rawBody, "X-Signature": `${timestamp}.${signature}` };
}

// Cliente: verifica firma y rechaza replays (timestamp > 5 min)
function verifyWebhook(rawBody, tstamp, sig, secret) {
  if (Math.abs(Date.now() / 1000 - Number(tstamp)) > 300) return false; // anti-replay
  const expect = crypto.createHmac("sha256", secret).update(`${tstamp}.${rawBody}`).digest("hex");
  return crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(expect));
}
```

Handler idempotente y tolerante a out-of-order: debounce por `event_id` y máquina de estados.

```js
async function onEvent(evt) {
  if (await redis.setnx(`events:${evt.id}`, 1, "EX", 86400) === 0) return; // idempotente
  // orden-independiente: ignorar "updated" si la entidad no existe aún
  if (evt.type === "order.updated" && !(await db.exists("order-" + evt.order_id))) return;
  await applyEvent(evt); // event-sourced state machine
}
```

---

## 6. Spec OpenAPI 3.x de ejemplo (fragmento)

`openapi: "3.1.0"` (alineado con JSON Schema 2020-12). La fuente de verdad; genera SDKs y valida runtime.

```yaml
openapi: "3.1.0"
info:
  title: Orders API
  version: "2017-05-25"
paths:
  /v1/charges:
    post:
      operationId: createCharge
      parameters:
        - name: Idempotency-Key
          in: header
          required: true
          schema: { type: string, maxLength: 255 }
        - name: X-API-Version
          in: header
          schema: { type: string }
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ChargeCreate"
      responses:
        "201":
          description: Charge created
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Charge" }
        "400":
          $ref: "#/components/responses/Problem"
components:
  responses:
    Problem:
      description: RFC 9457 problem details
      content:
        application/problem+json:
          schema:
            $ref: "#/components/schemas/Problem"
  schemas:
    Problem:
      type: object
      required: [type, title, status]
      properties:
        type: { type: string }
        title: { type: string }
        status: { type: integer }
        detail: { type: string }
        instance: { type: string }
```

---

## 7. Glosario extendido

- **REST / API resource:** estilo de recursos y verbos HTTP; stateless y cacheable.
- **GraphQL:** query language de un solo endpoint; `resolvers`, schema strong-type, introspectable.
- **gRPC:** RPC sobre HTTP/2 con Protobuf; 4 modos de streaming.
- **Webhook / callback:** push asíncrono del servidor al cliente; inversa del request/response.
- **Idempotencia / Idempotency-Key:** operación con efecto único pese a múltiples ejecuciones; clave UUID en mutaciones.
- **SET NX / lock distribuido:** primitiva Redis para exclusión mutua del procesamiento de un key.
- **Cursor / OFFSET:** paginación por marcador estable vs por salto posicional.
- **Problem Details (RFC 9457):** formato `application/problem+json` para errores.
- **Versioning por aislamiento / translation layer:** core en última versión + módulos de transform.
- **Sunset / Deprecated:** headers de deprecación machine-readable.
- **N+1 / over- / under-fetching:** patrones de ineficiencia en acceso a datos por API.
- **BFF (Backend-for-Frontend):** capa que adapta la API a un cliente específico.
- **Contract-first / OpenAPI-first:** el spec como fuente de verdad del contrato.