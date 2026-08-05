# Apéndice A — Módulo 11: Implementaciones de referencia

> Este apéndice complementa [11-Patrones-de-Diseño.md](11-Patrones-de-Diseño.md) con código concreto de los patrones clave en TypeScript/Node.js y Python, más una tabla de decisiones y un glosario extendido. Cada ejemplo prioriza *el problema que resuelve* sobre la sintaxis.

---

## 1. Strategy (pagos intercambiables) — TypeScript

El patrón que materializa *encapsulate what varies* + *program to an interface*. Añadir un método de pago no modifica el contexto (OCP).

```ts
// strategy.ts
interface PaymentStrategy {
  charge(amountCents: number, token: string): Promise<PaymentResult>;
}

class StripeStrategy implements PaymentStrategy {
  async charge(amountCents: number, token: string) {
    // ...llamada a Stripe
    return { provider: "stripe", status: "succeeded" };
  }
}
class AdyenStrategy implements PaymentStrategy {
  async charge(amountCents: number, token: string) {
    // ...llamada a Adyen
    return { provider: "adyen", status: "succeeded" };
  }
}

// Contexto: no conoce los concretos; recibe la estrategia por inyección
class Checkout {
  constructor(private strategy: PaymentStrategy) {}
  setStrategy(s: PaymentStrategy) { this.strategy = s; }  // intercambio en runtime
  async pay(amountCents: number, token: string) {
    return this.strategy.charge(amountCents, token);
  }
}
```

**Para añadir cripto:** `class CryptoStrategy implements PaymentStrategy` — sin tocar `Checkout`.

---

## 2. Decorator (retry + cache sobre un cliente HTTP) — TypeScript

Comportamiento añadido en runtime por composición, sin modificar el cliente. El orden de anidación importa.

```ts
// decorator.ts
interface HttpClient { get<T>(path: string): Promise<T>; }

class RealClient implements HttpClient {
  async get<T>(path: string): Promise<T> { /* fetch real */ }
}

// Decorator base
abstract class HttpClientDecorator implements HttpClient {
  constructor(protected inner: HttpClient) {}
  abstract get<T>(path: string): Promise<T>;
}

class CacheDecorator extends HttpClientDecorator {
  private cache = new Map<string, unknown>();
  async get<T>(path: string): Promise<T> {
    if (this.cache.has(path)) return this.cache.get(path) as T;
    const v = await this.inner.get<T>(path);
    this.cache.set(path, v);
    return v;
  }
}

class RetryDecorator extends HttpClientDecorator {
  constructor(inner: HttpClient, private retries = 3) { super(inner); }
  async get<T>(path: string): Promise<T> {
    for (let i = 0; ; i++) {
      try { return await this.inner.get<T>(path); }
      catch (e) { if (i >= this.retries) throw e; /* backoff + jitter (M10) */ }
    }
  }
}

// Uso: cache por fuera, retry en medio, cliente real adentro
const client = new RetryDecorator(new CacheDecorator(new RealClient()));
```

---

## 3. State (máquina de estados de un pedido) — TypeScript

Centraliza las transiciones válidas; una transición inválida lanza error en lugar de corromper estado.

```ts
// state.ts
type OrderStateName = "created" | "paid" | "shipped" | "delivered" | "cancelled";

// Tabla de transiciones válidas (inmutable y testeable)
const TRANSITIONS: Record<OrderStateName, OrderStateName[]> = {
  created:   ["paid", "cancelled"],
  paid:      ["shipped", "cancelled"],
  shipped:   ["delivered"],
  delivered: [],
  cancelled: [],
};

class Order {
  constructor(private state: OrderStateName = "created") {}
  transition(to: OrderStateName) {
    if (!TRANSITIONS[this.state].includes(to)) {
      throw new Error(`Transición inválida: ${this.state} → ${to}`);
    }
    this.state = to;
  }
  get current() { return this.state; }
}
```

**Equivalente en infra:** AWS Step Functions (M04) define máquinas de estado declarativas — la misma idea a nivel de workflow.

---

## 4. Chain of Responsibility (middleware HTTP) — TypeScript

Cada handler procesa o delega; añadir un handler no toca los existentes.

```ts
// chain.ts
type Ctx = { user?: string; scope?: string; ip: string };
type Next = () => Promise<void>;

abstract class Middleware {
  abstract handle(ctx: Ctx, next: Next): Promise<void>;
}

class Authn implements Middleware {
  async handle(ctx: Ctx, next: Next) {
    if (!ctx.user) throw new Error("401");
    await next();
  }
}
class Authz implements Middleware {
  async handle(ctx: Ctx, next: Next) {
    if (ctx.scope !== "admin") throw new Error("403");
    await next();
  }
}
class RateLimit implements Middleware {
  async handle(ctx: Ctx, next: Next) { await next(); } // token bucket (M10)
}

async function runPipeline(middlewares: Middleware[], ctx: Ctx) {
  let i = 0;
  const next = async () => {
    if (i < middlewares.length) await middlewares[i++].handle(ctx, next);
  };
  await next();
}
```

---

## 5. Factory Method (repositorios por config) — Python

Desacopla la creación del cliente del tipo concreto: `create_repo("postgres" | "dynamo")`.

```python
# factory.py
from abc import ABC, abstractmethod

class OrderRepository(ABC):
    @abstractmethod
    def get(self, order_id: str): ...

class PostgresOrderRepository(OrderRepository):
    def get(self, order_id: str): ...

class DynamoOrderRepository(OrderRepository):
    def get(self, order_id: str): ...

# Factory Method: decide qué concreto crear según config
def create_order_repository(kind: str) -> OrderRepository:
    if kind == "postgres":
        return PostgresOrderRepository()
    if kind == "dynamo":
        return DynamoOrderRepository()
    raise ValueError(kind)
```

---

## 6. Observer (eventos en memoria) — Python

El patrón detrás de pub/sub y de la programación reactiva (M03).

```python
# observer.py
class OrderService:
    def __init__(self):
        self._listeners = []

    def subscribe(self, listener):          # Observer: se registra
        self._listeners.append(listener)

    def notify_paid(self, order_id: str):
        for listener in self._listeners:    # notifica a todos
            listener.on_order_paid(order_id)

# Listener de ejemplo (notificador de emails)
class EmailNotifier:
    def on_order_paid(self, order_id: str): ...
```

---

## 7. Singleton → DI singleton-scoped (la alternativa moderna)

```ts
// di.ts (contorno de un contenedor DI)
class Container {
  private registry = new Map<string, unknown>();
  singleton<T>(key: string, factory: () => T): T {
    if (!this.registry.has(key)) this.registry.set(key, factory());
    return this.registry.get(key) as T;
  }
}
const di = new Container();
// una sola instancia del pool compartido, pero inyectable y testeable
const db = di.singleton("db-pool", () => createPool(process.env.DATABASE_URL));
// en tests: di.singleton("db-pool", () => fakePool());
```

---

## 8. Workflow Event (error-handling en EDA) — TypeScript

El procesador delega el error y pasa al siguiente mensaje (la responsividad no se degrada); un worker *repara* y *resubmite*.

```ts
// workflow-event.ts (concepto, abstrae el broker)
interface Message { id: string; payload: string; }

class TradeProcessor {
  constructor(
    private readonly broker: Broker,
    private readonly errorHandler: ErrorHandler, // workflow delegate
  ) {}

  async processMessage(msg: Message) {
    try {
      const [acct, side, symbol, shares] = msg.payload.split(",");
      const n = Long.parseLong(shares); // "8756 SHARES" → lanza aquí
      // ...colocar la orden
    } catch (e) {
      // DELEGACIÓN: el error se repara fuera; este procesador sigue con el siguiente mensaje
      await this.errorHandler.delegate(msg, e);
    }
  }
}

class TradeErrorHandler implements ErrorHandler {
  async delegate(msg: Message, e: unknown) {
    const fixed = msg.payload.replace(/ SHARES$/, ""); // CONTENCIÓN + REPARACIÓN
    await this.broker.resubmit({ ...msg, payload: fixed }); // RESUBMIT a la cola original
    // si no reparable → this.broker.sendToReviewQueue(msg) (dashboard humano + reply-to)
  }
}
```

**Puntos clave senior:** (1) el `TradeProcessor` **no** se bloquea en el error; (2) los mensajes reparados salen de secuencia → si el orden importa, cola temporal FIFO por contexto (cuenta); (3) el consumidor usa **client acknowledge mode** para no perder el mensaje si crashea.

---

## 9. Payloads de eventos: key-based vs data-based (y anemic events)

La granularidad del payload es un **espectro**, no un binario. El punto correcto evita *anemic events* (el consumidor no puede decidir).

```ts
// events.ts
// Key-based: mínimo acoplamiento, pero el consumidor no puede decidir sin consultar
type ProfileUpdatedKey = { event: "profile_updated"; customerId: string };

// Data-based: suficiente contexto para decidir (incluye before/after)
type ProfileUpdatedData = {
  event: "profile_updated";
  customerId: string;
  changed: { field: "address" | "phone"; before?: string; after: string };
};

// El consumidor puede decidir sin consultar el origen:
function onProfileUpdated(e: ProfileUpdatedData) {
  if (e.changed.field === "address") shippingService.update(e);   // decide
  // ...con solo { customerId } no habría podido (anemic event)
}
```

**Regla senior:** para *crear/borrar* un key-based basta; para *actualizar* incluye before/after (la BD típicamente no refleja el valor previo). El *Swarm of Gnats* es el extremo opuesto: demasiados eventos derivados *finos* por el mismo cambio (uno por campo) — consolida por **resultado del cambio**.

---

## 10. CQRS en código (paths de lectura y escritura separados) — TypeScript

```ts
// cqrs.ts
// Command (escritura): modelo normalizado, validaciones, transacción
class OrderCommandService {
  constructor(private repo: OrderWriteRepo, private bus: EventBus) {}
  async create(dto: CreateOrder) {
    const order = await this.repo.save(dto);      // write model (normalizado)
    await this.bus.publish("order_created", { id: order.id, ...dto });
  }
}

// Query (lectura): modelo desnormalizado/optimizado, sin lógica de escritura
class OrderQueryService {
  constructor(private readRepo: OrderReadRepo) {}  // read model, sync async (evento → denormalize)
  async findForDashboard(userId: string) {
    return this.readRepo.byUser(userId);           // joins/tablas planas ya hechas
  }
}
```

**Cuándo NO:** CRUD simple donde un modelo sirve a ambos paths — ahí CQRS es complejidad pura.

---

## 11. Sidecar / Service Mesh — configuración (concepto, Envoy/Istio)

El mesh implementa el *reuse operacional*: los concerns transversales (retry, circuit breaker, mTLS, observabilidad) viven en el sidecar, no en el código de dominio.

```yaml
# virtualservice.yaml (Istio) — comportamiento inyectado SIN tocar el servicio
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: payments
spec:
  hosts: [payments]
  http:
    - retries:            # retry operativo en el data plane
        attempts: 3
        perTryTimeout: 2s
        retryOn: connect-failure,5xx
      timeout: 10s
      route:
        - destination:
            host: payments
            subset: v2   # canary routing
```

**Trade-off senior:** el mesh añade un salto de red y requiere madurez operativa; lo justificas con muchos servicios y concerns transversales comunes — no para 3 microservicios.

---

## 12. Strangler Fig — routing progresivo (patrón de migración)

```ts
// strangler.ts — routing layer que migra feature por feature
type Feature = "orders" | "catalog" | "billing";

const MIGRATED: Feature[] = ["catalog"]; // cada feature nueva vive en el nuevo sistema

async function route(req: Request) {
  const feature = featureOf(req);
  if (MIGRATED.includes(feature)) return await newSystem.handle(req); // pieza nueva
  return await legacySystem.handle(req);                                // resto al legacy
}
// Cuando MIGRATED == todas las features → se retira el legacy (sin big-bang)
```

---

## 13. Tabla de decisión rápida (problema → patrón)

| Problema | Patrón | ¿Por qué? |
|---|---|---|
| Algoritmos intercambiables (pago, envío, descuento) | **Strategy** | Encapsula la variación tras una interfaz (OCP) |
| Añadir cache/retry/log a un cliente sin tocarlo | **Decorator** | Composición en runtime |
| Crear objetos sin acoplar al tipo concreto | **Factory Method / Abstract Factory** | Desacopla creación de uso |
| Construcción compleja paso a paso | **Builder** | Objetos inmutables + validación |
| Una única instancia de recurso compartido | **Singleton / DI singleton** | Control del recurso (cuidado con tests) |
| Interfaz incompatible de terceros | **Adapter** | Traduce al contrato esperado |
| Simplificar un subsistema complejo | **Facade** | Interfaz simple (API Gateway, SDK) |
| Control de acceso / deferir recurso costoso | **Proxy** | Intercepta sin modificar |
| Cadena de handlers que procesan o delegan | **Chain of Responsibility** | Middleware |
| Notificar cambios a muchos listeners | **Observer** | Pub/sub en memoria |
| Comportamiento según estado | **State** | Evita if/else gigantes, invariantes |
| Operación como objeto (jobs, undo) | **Command** | Encapsula, cola, retry |
| Flujo central con error-handling fuerte | **Orquestación** (Step Functions) | Estado central consultable |
| Escala/desacople de eventos | **Coreografía** (SNS/SQS) | Sin punto único, paralelismo |
| Reads y writes con características distintas | **CQRS** | Paths aislados y escalables |
| Concerns operativos en todos los servicios | **Sidecar / Service Mesh** | Plano transversal (ortogonal) |
| Un broker por dominio vs uno global | **Domain-Broker** | Aislamiento + escalabilidad (trade-off de costo) |
| Error en un flujo asíncrono sin usuario síncrono | **Workflow Event** (DLQ + reparación) | Delegación/containment/repair, responsividad intacta |
| ¿Cuánto dato lleva el evento? | **Payload key-based vs data-based** | Espectro: acoplamiento vs contexto (evita anemic events) |
| Demasiados eventos derivados finos | **Consolidar por resultado** (evitar Swarm of Gnats) | Granularidad por estado del cambio |
| Perder mensajes entre productor/broker/consumidor | **Event Forwarding** | Persistent queues + sync send, client ack, ACID/LPS |
| Coordinar transacciones distribuidas sin 2PC | **Saga** (choreographed/orchestrated) | Compensating actions |
| Migrar un legacy sin reescribir todo | **Strangler Fig** | Extracción gradual + routing layer |
| Cambiar coordinación por escala | **Orquestación → Coreografía gradual** | Estado central vs paralelismo (quanta) |

---

## 14. Glosario extendido

- **GoF:** "Gang of Four" — autores de *Design Patterns* (1994), 22 patrones clásicos.
- **Creacional / Estructural / De comportamiento:** las tres familias de patrones GoF (crear, componer, comunicar).
- **Encapsulate what varies:** separar lo que cambia de lo estable (compartimentos del barco).
- **Program to an interface:** depender de abstracciones, no concretos.
- **Favor composition over inheritance:** delegar comportamiento en objetos compuestos.
- **Estilo vs patrón:** topología con características asumidas vs solución contextualizada a un problema.
- **Particionado técnico vs dominio:** capas por capacidad vs por workflow/dominio (DDD).
- **Architecture quantum:** unidad de despliegue con independencia funcional.
- **Orquestación / Coreografía:** coordinación central vs eventos sin coordinador.
- **CQRS:** separación de caminos de lectura y escritura.
- **Sidecar / Service Mesh:** adjuntar concerns operativos en un plano transversal.
- **Domain-Broker / Single-Broker:** broker por dominio vs broker global (trade-off aislamiento vs descubrimiento).
- **Workflow Event:** patrón de error-handling en EDA (delegación, contención, reparación; resubmit a la cola original; revisión humana como fallback).
- **Anemic event:** evento derivado con contexto insuficiente para que el consumidor decida o continúe.
- **Swarm of Gnats (antipatrón):** demasiados eventos derivados finos por un mismo cambio; se evita consolidando por resultado del cambio.
- **Event Forwarding:** técnicas para prevenir pérdida de datos en EDA (persistent queues + synchronous send, client acknowledge mode, ACID commit + LPS).
- **Saga:** secuencia de transacciones locales + compensating actions para transacciones distribuidas sin 2PC.
- **Strangler Fig:** migración gradual de un legacy, feature por feature, con un routing layer; el legacy se retira al final.
- **Inverse Conway Maneuver:** reestructurar equipos para lograr la arquitectura deseada.
- **Big Ball of Mud:** antipatrón de estructura caótica sin fronteras.
- **Leyes de la Arquitectura:** todo es trade-off; el "por qué" importa más que el "cómo"; las decisiones viven en un espectro.