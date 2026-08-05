# Apéndice A — Módulo 10: Implementaciones de referencia

> Este apéndice complementa [10-Escalabilidad-y-Resiliencia.md](10-Escalabilidad-y-Resiliencia.md) con código y configuración concretos: cache-aside con stampede protection, retry con backoff + jitter, circuit breaker, bulkhead, token bucket rate limiter, autoscaling por latencia en IaC y glosario extendido.

---

## 1. Cache-aside con stampede protection (Node.js)

El patrón lazy con *single-flight* para evitar el *thundering herd* cuando un item caliente expira: un solo proceso recalcula, el resto espera o usa stale.

```js
// cache-aside.js
const pending = new Map(); // single-flight: key -> promise

async function getCached(redis, key, ttl, loader) {
  const hit = await redis.get(key);
  if (hit != null) return JSON.parse(hit);

  // Stampede protection: solo un loader recalcula por key
  if (pending.has(key)) return pending.get(key);

  const p = (async () => {
    try {
      const value = await loader();
      await redis.set(key, JSON.stringify(value), "EX", ttl + jitter(ttl)); // TTL con jitter
      return value;
    } finally {
      pending.delete(key);
    }
  })();
  pending.set(key, p);
  return p;
}

const jitter = (ttl) => Math.floor(Math.random() * ttl * 0.2); // ±20%

// Uso
const price = await getCached(redis, `price:${sku}`, 60, () => db.getPrice(sku));
```

Alternativa *stale-while-revalidate*: ante un hit expirado, devuelve el valor viejo y dispara un refresh en background (`SET ... NX` guarda el "lanzador").

---

## 2. Retry con backoff exponencial + jitter

```js
async function retry(fn, { maxRetries = 3, base = 100, max = 2000 } = {}) {
  for (let attempt = 0; ; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt >= maxRetries || !isTransient(err)) throw err; // solo transitorios
      const backoff = Math.min(max, base * 2 ** attempt);          // 100, 200, 400
      const sleep = backoff + Math.random() * backoff;             // + jitter ±100%
      await new Promise((r) => setTimeout(r, sleep));
    }
  }
}

const isTransient = (e) =>
  e?.status >= 500 || e?.code === "ECONNRESET" || e?.code === "ETIMEDOUT" ||
  (e?.status === 429 && e?.headers?.["retry-after"] != null);
// Con Retry-After: respetar el header del servidor en lugar del backoff propio.
```

**Retry budget global** (evita que un fallo multiplique la carga):

```js
// Contador atómico en Redis; si el sistema ya está en "modo retry", no añade más
async function canRetry(redis) {
  const n = await redis.incr("global:retry_budget");
  await redis.expire("global:retry_budget", 60);
  return n <= 100; // máx. 100 retries por minuto en todo el sistema
}
```

---

## 3. Circuit breaker (implementación minimalista)

Estados Closed / Open / Half-Open con fallback. La llamada protegida *falla rápido* cuando el circuito está abierto.

```js
class CircuitBreaker {
  constructor({ failureThreshold = 5, windowMs = 10_000, resetMs = 30_000, fallback } = {}) {
    Object.assign(this, { failureThreshold, windowMs, resetMs, fallback });
    this.state = "CLOSED";
    this.failures = [];
    this.openedAt = 0;
  }
  async call(fn) {
    if (this.state === "OPEN") {
      if (Date.now() - this.openedAt >= this.resetMs) this.state = "HALF_OPEN"; // prueba
      else return this.fallback(); // fail fast + fallback
    }
    try {
      const result = await fn();
      if (this.state === "HALF_OPEN") this.reset(); // la prueba pasó → cerrado
      this.failures = this.failures.filter((t) => t > Date.now() - this.windowMs);
      return result;
    } catch (err) {
      this.failures.push(Date.now());
      if (this.failures.length >= this.failureThreshold) {
        this.state = "OPEN";
        this.openedAt = Date.now();
      }
      return this.fallback(err);
    }
  }
  reset() { this.state = "CLOSED"; this.failures = []; }
}

const paymentBreaker = new CircuitBreaker({
  failureThreshold: 5, windowMs: 10_000, resetMs: 30_000,
  fallback: () => ({ status: "degraded", source: "cache" }), // graceful degradation
});
```

En la práctica usa Resilience4j (JVM), Polly (.NET) o el breaker del service mesh (Istio `DestinationRule`/`VirtualService`).

---

## 4. Bulkhead: aislamiento de recursos por dependencia

Thread/connection pools **separados por dependencia** para que una saturada no devore las demás.

```js
// Node.js: límites por dependencia
const { pool } = require("generic-pool");

const pools = {
  payments:  pool({ create: () => makeClient("payments"), max: 10 }),   // 10 conns
  inventory: pool({ create: () => makeClient("inventory"), max: 10 }),  // 10 conns
  catalog:   pool({ create: () => makeClient("catalog"), max: 50 }),    // 50 conns
};

// Si `payments` agota sus 10 conexiones, NO toca a catalog/inventory
async function withBulkhead(name, fn) {
  const conn = await pools[name].acquire();
  try { return await fn(conn); }
  finally { pools[name].release(conn); }
}
```

Equivalente en infra: **pods/instancias separados** por dependencia crítica, o *partitions* en el mesh. La garantía: la explosión de un bucket no quema los otros.

---

## 5. Rate limiter distribuido: token bucket con Redis

Bucket con `capacity` tokens y recarga a `rate` por segundo. **Distribuido**: el contador es compartido (no se multiplica por instancia).

```lua
-- token_bucket.lua
local key   = KEYS[1]        -- "rl:" .. clientId .. ":" .. resource
local rate  = tonumber(ARGV[1])
local burst = tonumber(ARGV[2])
local now   = tonumber(ARGV[3])

local data  = redis.call("HMGET", key, "tokens", "ts")
local tokens = tonumber(data[1]) or burst
local ts     = tonumber(data[2]) or now

local refill = math.floor((now - ts) / 1000) * rate   -- tokens ganados desde último update
tokens = math.min(burst, tokens + refill)

if tokens >= 1 then
  redis.call("HSET", key, "tokens", tokens - 1, "ts", now)
  redis.call("EXPIRE", key, 300)
  return 1   -- allow
else
  return 0   -- deny → 429
end
```

```js
// Gateway / middleware
const allowed = await redis.eval(tokenBucketScript, 1, `rl:${client}:api`, 100, 200, Date.now());
if (!allowed) {
  return res.status(429)
    .set("Retry-After", 1)
    .set("X-RateLimit-Limit", 200)
    .set("X-RateLimit-Remaining", 0)
    .end();
}
```

Sliding window (más preciso, sin ráfaga de borde): `ZADD key now id` + `ZREMRANGEBYSCORE key -inf now-window` + `ZCARD` + `EXPIRE`.

---

## 6. Autoscaling por latencia en IaC (AWS)

Escalar por **latencia** en lugar de CPU, con cooldown para evitar thrashing y un **programado** para picos conocidos.

```yaml
# CloudFormation: ASG con target tracking por latencia p95
OrdersAutoScalingGroup:
  Type: AWS::AutoScaling::AutoScalingGroup
  Properties:
    MinSize: 2
    MaxSize: 20
    DesiredCapacity: 4
    Cooldown: 300                       # evita oscilación
    HealthCheckGracePeriod: 120
    TargetGroupARNs:
      - !Ref OrdersTargetGroup          # el ALB drena conexiones al escalar (deregistration delay)
    LaunchTemplate: { LaunchTemplateId: !Ref OrdersLaunchTemplate, Version: !GetAtt OrdersLaunchTemplate.LatestVersion }

LatencyPolicy:
  Type: AWS::AutoScaling::ScalingPolicy
  Properties:
    AutoScalingGroupName: !Ref OrdersAutoScalingGroup
    PolicyType: TargetTrackingScaling
    TargetTrackingConfiguration:
      PredefinedMetricSpecification:
        PredefinedMetricType: ALBRequestCountPerTarget  # o custom: p95 latency
      TargetValue: 500                                  # < 500 ms

ScheduledScaleUp:
  Type: AWS::AutoScaling::ScheduledAction
  Properties:
    AutoScalingGroupName: !Ref OrdersAutoScalingGroup
    MinSize: 15
    MaxSize: 30
    DesiredCapacity: 20
    Recurrence: "0 12 * * *"            # pico de venta a las 12:00 UTC
```

---

## 7. Timeouts + deadline propagation (contexto)

```js
// Llamada con deadline que se propaga: cada dependencia recibe el tiempo restante
async function withDeadline(parentDeadlineMs, fn) {
  const now = Date.now();
  const remaining = parentDeadlineMs - now;
  if (remaining <= 0) throw new DeadlineExceededError();
  return withTimeout(fn, remaining); // usa el resto del presupuesto, no 30 s fijos
}

// Uso en cadena: pagos → inventario
async function checkout(req) {
  const budget = req.deadline;                       // presupuesto total del request (ej. 2000 ms)
  const charge = await withDeadline(budget, () => payClient.charge());        // usa lo que quede
  await withDeadline(budget, () => inventory.reserve());                      // y vuelve a comprobar
}
```

Reglas: **connect < read < total** por dependencia; el contexto se pasa hacia abajo para que la suma nunca exceda el presupuesto; si algo se pasa del deadline → fallback o error 504 rápido, no un cuelgue.

---

## 8. Glosario extendido

- **Scalability / Scale up / Scale out:** capacidad de crecer; vertical (recursos) vs horizontal (nodos).
- **Stateless:** instancias intercambiables; el estado vive fuera (Redis/BD/S3).
- **Load balancer / health check / draining:** distribución de tráfico, detección de nodos muertos, y espera de requests en vuelo al dar de baja.
- **Autoscaling / cooldown / target tracking:** escalado automático por métrica; enfriamiento anti-thrash; objetivo de métrica (ej. p95).
- **Throughput / Latency / Percentiles / Tail latency:** tasa completada; tiempo por op; p50/p95/p99/p999; la cola lenta percibida.
- **Ley de Little (L = λ·W):** requests en vuelo = tasa × tiempo.
- **Caché / cache-aside / write-through / stampede / TTL-jitter:** capas y patrones de caché; la avalancha al expirar.
- **CDN / edge:** caché geográfica en el borde.
- **Read replica / lag / read-your-writes:** copias de lectura; el retraso asíncrono y su manejo.
- **Backpressure / load shedding:** control de flujo; rechazo deliberado bajo sobrecarga.
- **Rate limiting / token bucket / sliding window:** límite decisivo de flujo.
- **Circuit breaker (Closed/Open/Half-Open) / fallback:** corte de una dependencia enferma + respuesta degradada.
- **Retry / backoff / jitter / retry storm / retry budget:** reintentos seguros y su peligro.
- **Timeout / deadline propagation:** límites temporales por llamada y su suma en cadenas.
- **Bulkhead:** aislamiento de recursos por dependencia.
- **Graceful degradation / fault tolerance:** degradación útil; operación continua ante fallos.
- **Chaos engineering / blast radius / game day:** inyección de fallos controlada y con hipótesis.