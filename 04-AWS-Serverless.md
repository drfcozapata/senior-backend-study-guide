# Módulo 04 — AWS Serverless (Lambda, API Gateway, EventBridge, Step Functions, Cognito)

> **Objetivo profesional:** Dominar AWS Serverless no como "servicios" sino como **universo de trade-offs**. Entender cuándo Lambda es la respuesta correcta y cuándo es un error, cómo diseñar APIs con API Gateway sin perder performance, cómo orquestar workflows distribuidos con Step Functions sin que se conviertan en "código spaghetti en JSON", cómo EventBridge se diferencia de SNS/SQS, y cómo Cognito resuelve identidad sin construir tu propio IdP.

> **Este módulo es el puente entre la teoría (Módulos 01-03) y la infraestructura real.** Se apoya en [Módulo 02](02-Microservicios-y-DDD.md) (patrones de comunicación), [Módulo 03](03-Event-Driven-Architecture.md) (EventBridge, SQS, SNS, Saga) y se conecta con [Módulo 05](05-Bases-de-datos-distribuidas.md) (DynamoDB como persistencia natural de Lambda), [Módulo 06](06-Infraestructura-como-Codigo.md) (Terraform/CloudFormation para desplegar todo esto), y [Módulo 07](07-Observabilidad.md) (CloudWatch, X-Ray para traces).

---

## Introducción

AWS Serverless no es "no hay servidores" — es que **no gestionas servidores**. El servidor existe, pero AWS lo abstracta: tú subes función, ellos gestionan capacidad, escalado, parches y disponibilidad. El precio es una serie de trade-offs que, mal entendidos人民, producen sistemas mucho peores que los que reemplazan.

**Ventajas reales (con context):**

- Zero scaling logic: de 0 a thousands de invocaciones concurrentes sin configuración.
- Pay-per-use: pagas por milisegundo de ejecución y GB-segundo, no por uptime.
- Integración nativa con el ecosistema AWS (DynamoDB, S3, SNS, SQS, CloudWatch, X-Ray).
- Menor superficie operativa: no patches, no AMIs, no auto-scaling groups.

**Desventajas reales (que los tutoriales omiten):**

- **Cold starts:** la primera invocación de una función "dormida" incluye inicialización del runtime (100ms - 5000ms dependiendo del runtime y dependencias). En Node.js/Go son aceables; en JVM/Java son dolorosos a menos que uses SnapStart.
- **Límites estrictos:** 15 min max execution time, 10GB RAM max, payload limits (256KB para HTTP via API Gateway, 256KB para SQS, 6MB para invocación síncrona directa).
- **Vendor lock-in profundo:** el patrón Lambda + API Gateway + DynamoDB + EventBridge es difícil de portar a GCP/Azure sin reescritura.
- **Costo a escala:** a alto volumen constante (ej: 24/7 con 5000 req/s), EC2/ECS con Reserved Instances suele ser más barato que Lambda.
- **Debugging y testing local:** requiere herramientas especiales (SAM CLI, LocalStack, Docker emulación).

**Cuándo es correcto:**

- Eventos y triggers (S3 upload → thumbnail, DynamoDB stream → proyección).
- APIs con tráfico variable/impredecible.
- Glue jobs yETLLight.
- Prototipos y MVPs donde speed-to-market manda.

**Cuándo es incorrecto:**

- Workloads constantes 24/7 a alto throughput (más barato contendedores/EC2).
- Procesos con estado interno complejo que necesitan conexiones persistentes a múltiples backends (connection pooling agresivo).
- Aplicaciones que requieren más de 15 minutos de ejecución continua.
- Sistemas donde cold start de 3-5s es inaceptable (trading,HJ gaming de alta frecuenta).

---

## Conceptos

### AWS Lambda: más que "funciones"

**Qué es realmente:**
Un servicio de cómputo serverless donde:

- Subes código (o contenedor hasta 10GB).
- AWS ejecuta tu función en respuesta a _events_ (HTTP, S3, DynamoDB Streams, SQS, etc.).
- Escala horizontalmente ejecutando múltiples instancias en paralelo (hasta un límite regional por defecto de 1000 concurrent executions, ampliable via support ticket).
- Tú defines todo el entorno: runtime (Node.js 20.x, Python 3.12, Java 21, .NET 8, Go 1.x, Rust custom runtime, o container con OCI image).

**Anatomía de ejecución:**

```
Evento (HTTP, SQS, S3) → Runtime (proceso con tu código + runtime manager)
                        ↘ Handler(event, context)
                        ↘ return response / error
```

**Configuración clave:**

- **Memoria:** 128MB a 10,240MB (10GB). CPU granular: ~1 vCPU por 1.7GB. A más memoria, proporcionalmente más CPU; la política de "aumentar memoria" es la única forma de aumentar CPU.
- **Timeout:** default 3s, máximo 900s (15 min).
- **Concurrency:** reservado vs. proveedor. Reserved concurrency limita a una función específica (esto resuelve el problema de "un servicio acaparando la cuenta").
- **Layers:** paquetes de librerías compartidas entre funciones (evitando duplicar dependencias en cada zip).
- **Extensions:** procesos paralelos que se inician con Lambda para agentes (monitoring, tracing, secrets refresh).

### Cold starts detallados

Una función Lambda tiene dos fases de inicialización:

1. **Init (cold start):** contenedor SO + runtime + dependencias (npm install, mvn shaded jar, etc.) cargan. Esto ocurre una vez per instancia activa.
2. **Invoke (warm start):** el runtime ya está listo; Lambda reutiliza la instancia para próximas invocaciones.

**Mitigaciones:**

- **SnapStart (Java 11+):** snapshot JVM precargada. Inicia en ~35ms. No disponible para Node/Python nativamente.
- **Provisioned Concurrency:** AWS mantiene N instancias precalentadas. Costo adicional, pero elimina cold starts en rutas críticas. VERSUS reserved concurrency: provisioned es por performance, reserved es por control.
- **Librerías nativas optimizadas:** usar `esbuild` para bundle Node.js en un solo archivo, evitando cold starts por análisis de múltiples imports.

### Eventos y Fuentes de invocación

Lambda se invoca por:

- **HTTP/REST:** API Gateway, ALB (Application Load Balancer con Lambda como target).
- **Messaging:** SQS (polling batch), SNS (publish), EventBridge (rules).
- **Streaming:** DynamoDB Streams (CDC), Kinesis Data Streams.
- **Storage:** S3 PutObject, CloudWatch Events (cron).
- **Custom:** IoT Core, Cognito Triggers, Alexa Skills, etc.

**Control de errores por fuente:**

- **SQS:** Lambda poll hasta 10 mensajes. Si falla, el batch se devuelve a la cola. Lambda compiler retry hasta el maxReceiveCount de la queue.
- **DynamoDB Streams:** process records en batch. Si falla, Lambda repara el **shard completo** (head-of-line blocking en el stream).
- **Sync (HTTP/ALB):** el caller espera respuesta. Si Lambda falla, API Gateway devuelve 502.

### API Gateway: más que "un proxy"

API Gateway es un servicio managed de alto rendimiento para crear, publicar, mantener APIs REST/HTTP/WebSocket. Realiza dos funciones principales:

1. **Edge:** recepción de peticiones (CORS, autenticación, validación, rate limiting, caching).
2. **Integration:** conectar con backend (Lambda, HTTP, AWS services, mock).

**Tipos:**

- **REST API:** la clásica. Payload transformation con VTL (Velocity Template Language), policies, linked resources.
- **HTTP API:** más ligero, más barato (70% cost reduction vs REST), no soporta algunas funciones avanzadas (VTL, API Keys, usage plans nativos — se pueden hacer con authorizers).
- **WebSocket API:** para conexiones persistentes (chat, notificaciones push).

**Funcionalidades avanzadas:**

- **Caching:** respuestas cacheables con TTL, invalidación automática o manual. Reduces backend load.
- **Throttling:** tokens bucket por usuario o global, pará controlar rafagas.
- **API Keys y Usage Plans:** no son para seguridad (muchos creen esto; es para metering y cotas).
- **Custom Domain + ACM:** certificados SSL/TLS y dominios bonitos para APIs.
- **Canary releases:** dividen tráfico entre versiones (ej: 10% a nueva versión, 90% a la anterior) antes de promover.

**Trade-offs de diseño:**

- API Gateway + DynamoDB directo (integration tipo `AWS service proxy`): atractivo por rapidez, pero pierdes lógica de negocio centralizada y es hardcobre debuggear. Úsalo brillosamente para APIs CRUD simples.
- Lambda como backend: lógica centralizada para autenticación, enriquecimiento, transforms, validation.

### EventBridge: bus de eventos serverless con powers

EventBridge es más que SNS 2.0. Características diferenciadoras:

- **Schema Registry:** catálogo centralizado de contratos de eventos. Auto-discovery de eventos publicados: EventBridge infiere el esquema y lo almacena. Facilita a otros equipos consumir eventos consistentemente.
- **Event Pipes:** conexión directa source→target (SQS→EventBridge→Step Functions, API Gateway→Lambda). Incluye filtering, transformación y enrichment sin Lambda intermedia.
- **Scheduler:** subnservicio para invocar targets en cron (reemplaza CloudWatch Events).
- **Cross-account/cross-region:** event buses pueden estar en cuenta A, región us-east-1, con reglas que envían a target en cuenta B.

**CUando usar cada uno:**
| Escenario | AWS Service |
|---|---|
| Notificación simple a múltiples suscriptores | SNS |
| Trabajo pesado desacoplado | SQS |
| Eventos con routing por atributos | EventBridge |
| Streams, replay, high-volume continuo | MSK / Kinesis Data Streams |

**Ejemplo con EventBridge:**

```json
{
  "source": ["myapp.orders"],
  "detail-type": ["OrderCreated"],
  "detail": {
    "customerId": ["c-456"],
    "total": [{ "numeric": [">", 100] }]
  }
}
```

Una regla puede matchear y enviar solo pedidos del customer c-456 con total > 100 a un Step Function de "VIP approval".

### Step Functions: orquestación serverless

Step Functions coordina múltiples servicios AWS en workflows business con state machines definidas en **Amazon State Language (ASL)**, que es JSON.

**Tipos de workflow:**

- **Standard:** max 1 año de duración, exactly-once processing, más barato para ejecuciones largas.
- **Express:** max 5 minutos, at-least-once, alto volumen (processing >100K eventos/seg), mucho más económico para duración corta.

**Ventaja técnica:** state machine _no corre código_, solo coordina. Cada `Task` state invoca una API (Lambda, DynamoDB, SQS, ECS Activity) y espera resultado. No mantienes conexiones persistentes.

**Patterns clave:**

- **Saga orquestada:** como vimos en [Módulo 03](03-Event-Driven-Architecture.md), perfección para: coordinar pasos de pedidos, patient discharge, billing cycles.
- **Parallel:** ejecuta ramas concurrentes (enriquecer pedido + verificar fraud + log interaction concurrently).
- **Choice:** decisiones basadas en input de la Task anterior.
- **Map:** iterar sobre arrays (process 1000 S3 keys como batch concurrente).
- **Wait:** pausar entre acciones sin llamar innecesariamente services.

**Idempotencia:** Step Functions suporta integraciones idempotentes (`$.Retry` con backoff, `ResultPath`, `Parameters` para earthing de inputs entre steps).

### Amazon Cognito: identidad sin infraestructura

Cognito soluciona **autenticación** y **director de usuarios** sin que construyas tu propio sistema. Es crítico entender sus componentes para no confundirlos:

- **User Pools:** directorio de usuarios completo (registro, sign-in, reset password, MFA, OAuth 2.0 tokens para consumo "user-level"). Emite JWTs estándar.
- **Identity Pools (Federated Identities):** paran acceso a AWS resources desde terceros (Google, Facebook, OpenID Connect, SAML). Genera credenciales IAM temporales.
- **Triggers:** Lambda functions que se ejecutan en ciertos puntos del credential flow (pre_sign_up, post_confirmation, etc.) para personalización.

**JWT de User Pools:** para applicaciones web/móvil típicas, el `id_token` y `access_token` emitidos por Cognito bastan para autenticar requests en API Gateway con authorizers nativos.

---

## Arquitectura

### 1. Diseño de aplicación serverless típica

```
┌─────────┐    ┌─────────────┐    ┌────────────┐    ┌────────────┐    ┌─────────┐
│ Client  │───▶│  Route 53   │───▶│  API GW    │───▶│ Lambda     │───▶│DynamoDB │
│(Web/SPA)│    │ + CloudFront│    │ (REST/HTTP)│    │ Fn (Auth)  │    │ (NoSQL) │
└─────────┘    └─────────────┘    └────────────┘    └────────────┘    └─────────┘
                                         │
                                         ▼
                                   ┌────────────┐
                                   │ EventBridge│
                                   │ (Event bus)│
                                   └────────────┘
                                         │
          ┌─────────┬──────────┬───────────┬───────────┐
          ▼         ▼          ▼           ▼           ▼
      ┌───────┐ ┌───────┐ ┌─────────┐ ┌─────────┐
      │ Lambda│ │ Lambda│ │   Step  │ │ Step Fn │
      │(Email)│ │(Image)│ │Functions│ │ │(Saga) │ │(DataETL)│
      └───────┘ └───────┘ └─────────┘ └─────────┘ └─────────┘
```

**Puntos de diseño:**

- Edge: CloudFront para CDN, Route 53 para DNS with health checks.
- API: HTTP API basta si no necesitas VTL transformations. REST cuando necesitas API Keys usage by plan.
- Lambdas: pequeñas, single purpose. Use context memoization (como apostillar en clase BD EN PV singleton).
- Data: DynamoDB para key-value y documentdb si acceso patters son simples. Amplifier por aggregate by features.
- Events: EventBridge con Schema Registry enables discovery. Separar event bus por dominio.
- Async procesamiento: SQS queue toLambda (work processors). Para streams+replay: Kinesis / MSK con consumers Lambda.

### 2. Authentication y Authorization flow

```
1. Cliente -> Cognito User Pool (Sign in)
   -> Retorna JWT (id_token + access_token + refresh_token)

2. Client calls API Gateway con `Authorization: Bearer <access_token>`

3. API Gateway Cognito Authorizer valida firma, expiración, scopes.

4. API GW inyecta claims ($context.authorizer.claims.sub) y los pasa a Lambda.

5. Lambda decide authorization: "usuario X puede acceder a sus own orders",
   using identity claim `sub` for data filtering.

6. Si expira: cliente usa refresh_token en Cognito -> new tokens.
```

Service-to-service entre Lambdas: usar `ClientCredentials` flow. En event processing, la emisión de nuevos tokens se hace con IAM roles (Execution Role) para invocar otros servicios AWS sin hardcode.

### 3. Flujo de agregación asíncrona con SQS+Step Functions

**Pattern:** trabajo pesado secuencial pero sin timeout de Lambda.

```
API Gateway HTTP
   └─> Lambda Fn_A (Split order into tasks)
      └─> SQS con Task messages (individual units)
         └─>[Trigger] Lambda Fn_B (Process each task, updates DynamoDB)
            └─> DynamoDB: Status updated

Cuando DynamoDB stream registra "all tasks COMPLETED":
   └─> EventBridge rule triggers Step Functions
        └─> Saga final (send email, issue invoice, register metrics)
             -> [Tasks states as they complete...]
```

### 4. WebSocket with API Gateway + Lambda

Para chat real-time:

- Cliente mantiene WebSocket connection (`$connect`, `$disconnect`, `$default` lambdas).
- Dialogos pub/sc events van a EventBridge.
- Reglas enrutan a Lambda encargada de `postToConnection`.

Estos patrones reemplazan to Simple WebSocket servers. Common gotcha: el `$connect` handler must store mapping between WebSocketID and userId (usually in DynamoDB with TTL).

---

## Internals: Lambda profundidad

### Runtime execution y estadísticas

HTTP-trigger:

```python
import json
def lambda_handler(event, context):
    # Accessing API Gateway proxy format v2.0
    body = json.loads(event.get('body', '{}'))
    return {
      "statusCode": 200,
      "headers": {"Content-Type": "application/json"},
      "body": json.dumps({"message": "ok"})
    }
```

AWS genera CloudWatch Logs con:

- `START` / `END` / `REPORT` (duration, billed duration, memory size, init duration).
- Metricas: Invocations, Duration, Errors, Throttles, ConcurrentExecutions.

**Memory optimization pattern:** With 1GB memory (≈0.58 vCPU). Benchmark fluorescencia between 512MB (single core) vs 1792MB (1.05 cores): a veces duplicar memoria halucinates cold start duration for CPU-bound fixture initialization (JS bundling, Java classloading).

**Provisioned Concurrency example:**

```yaml
ProvisionedConcurrencyConfig:
  ProvisionedConcurrentExecutions: 10
```

Mantiene 10 Lambdas precalentadas (billed as always running). Mesaure with CloudWatch `ProvisionedConcurrencyInvocations`.

**Cold start mitigations in practice:**

1. Node.js: use `esbuild` or `webpack` to bundle everything to single file.
2. Java: SnapStart.
3. Python: keeping `requirements.txt` minimal region specific runtime caching.
4. Arquitectura: middleware-heavy endpoints on API GW instead of Lambda where possible.

### Límites que hay que conocer

Payload limits (per invocation):

- API Gateway HTTP: 10MB body, 10KB header size total.
- SQS message: 256KB max.
- SNS message: 256KB per publish.
- EventBridge event: 256KB per event.
- S3 triggered Lambda: data event record only includes metadata (max ~6KB per event item), content fetched separately.
- Lambda direct sync invoke: 6MB request/response.

Concurrency:

- Default regional: 1000 concurrent executions. Reserved concurrency: up to ~60000 per account across all functions; you can reserve per function (e.g., critical fn gets 50).

### Patrón de inyección de dependencias y testabilidad

Lambda functions son complicadas de testear si metes todo en la misma función. Solución: Hexagonal + DI.

```typescript
// Handler.ts — Entry Point (Infra)
export const handler = async (event: APIGatewayProxyEvent) => {
  const repo = new DynamoOrderRepository();
  const useCase = new CreateOrderUseCase(repo);
  return useCase.execute(JSON.parse(event.body));
};

// createOrderUseCase.ts — Application (Domain)
export class CreateOrderUseCase {
  constructor(private repo: OrderRepository) {}

  async execute(input: CreateOrderDTO): Promise<Result<string>> {
    const order = Order.create(input);
    await this.repo.save(order);
    await this.publishEvents(order.events);
    return ok(order.id);
  }
}
```

Cargamos el repositorio real en producción, in-memory fake en tests — el use case no sabe qué está usando.

---

## Patrones

### Pattern: Fan-out con SNS + SQS

```text
EventBridge (orders.events)
   |
   └─→ SNS Topic "order-events"
         ├─→ SQS "notifications" → Lambda → Email
         ├─→ SQS "analytics"     → Lambda → Redshift/S3
         └─→ SQS "erp-sync"      → Lambda → SAP/HTTP
```

Esto combina routing (EventBridge) con desacoplamiento (SNS) y backpressure/retries (SQS). Es el patrón estándar para resultado distribución de eventos.

### Pattern: Lambda Power Tuning

Herramienta de AWS Labs que ejecuta función con diferentes configuraciones de memoria y mide costo/duration trade-off. Output recomienda optimization point.

Ejemplo típico de salida:

```
Memory  Cost Duration
128     $0.0002  15s
512     $0.0006   4s
1024    $0.0011   2s
2048    $0.0021   1.1s
```

A veces 2x memoria reduce duration by half → costs stay same, UX improved.

### Pattern: Idempotency Layer

Pattern aplicado a Lambda que procesa SQS:

```typescript
const DEDUP_TABLE = process.env.DEDUP_TABLE;

async function processRecord(record) {
  const messageId = record.messageId;
  const result = await dedupDB.insert({ PK: `MSG_${messageId}`, status: 'PROCESSING' },
    { ConditionExpression: 'attribute_not_exists(PK)' });

  if (result.alreadyExists) return; // at-least-once safety

  try {
    await businessLogic(record.body);
    await dedupDB.update({ ... , status: 'DONE' });
  } catch (err) {
    await dedupDB.update({ ..., status: 'FAILED', error: err.message });
    throw err; // Retries via SQS visibility timeout
  }
}
```

Store last processed rollover from DynamoDB stream (sequence number) to prevent reprocessing on consumer lag rebalance.

### Pattern: CORS handling

Don’t rely on Lambda to return CORS headers unless debug. Enable CORS in API Gateway directly (OPTIONS preflight si aplica). For HTTP API, simple configuration; REST API requires integration response mappings based on origin map.

---

## Casos Reales

### Caso 1: Migración de monolito a serverless en producción (E-commerce)

Un retailer migró su checkout desde Node monolito a Lambda + API Gateway + DynamoDB.

- **Altas lambdas:** `checkout-handler` para starts transactions, `payment-callback` para external payment confirmations, `inventory-reserve` para stock moves.
- **Resultado:** Deploy con 0 downtime, escala a BlackFriday (10x traffic), costes reducidos ~40% vs EC2 persistente, pero debugging inicial complejo hasta establecer(correlation ID + X-Ray).
- **Lección:** Authentication via Cognito User Pools redujo código by ~60%, no obstante tuvieron que migrar múltiples roles custom a User Pool Groups + IAM conditional policies.

### Caso 2: Procesamiento masivo con Map State (Fintech)

Una fintech debía calcular daily positions para ~10,000 accounts. Monolitica tardaba 45 minutos con prohibición de downtime.

Solución serverless con Step Functions Map:

- **Distributed Map State:** input array de IDs processed in parallel (maxConcurrent: 500, Batch input to avoid memory depletion).
- Cada "Task" invoques a Lambda which fetch account data, compute position, write result.
- `ResultWriter` in Step Functions EMITS summary to S3 for archival and analytics.
- **Time:** 45m → 4m with cost ~2.3x of single Lambda same duration (paralelism "pays more but completes faster" trade-off acceptable).

### Caso 3: WebSockets + Push Notifications para field teams

Empresa logística needed live dispatch updates on drivers app. Tradicional: WebSocket server behind ELB + connection state in Redis (~$1200/mo, horizontal scaling pain).

Solución: API Gateway WebSocket + Lambda + DynamoDB.

- `$connect` lambda saves `connectionId -> driverId`.
- Dispatcher publishes to SNS when new assignment.
- Fan-out Lambda sends POST to `https://{domain}/{stage}/@connections/{connectionId}` via API Gateway Management API.
- **Cost:** ~180/mo, automatic scaling, no stateful好景不长 servers. Management API per-attempt cost balances out.

**Crítico lesson:** Used TTL on DynamoDB connection records to auto-purge stale connectionIds. Middleware authorizer refreshed socket auth token every 15 minutes to avoid long-lived tokens compromise.

---

## Laboratorio

### Lab 1: Serverless CRUD API con DynamoDB

Task: Build a minimal todo API using HTTP API Gateway + Lambda + DynamoDB.

1. Bookmark POST /todos, GET /todos, PUT /todos/:id, DELETE /todos/:id.
2. CORS enabled.
3. Unit test the use case with mocks.
4. Integration test with SAM Local or LocalStack.

**Deliverables:** ZIP deployable via AWS SAM template (infra as code preview — Módulo 06).

### Lab 2: Order processing with Saga via Step Functions

Implement in TypeScript/JavaScript:

- Step Function `OrderSaga` con tambien:
  - `ValidateOrder` (Task → Lambda)
  - `ReservePayment` (Task → Lambda → Retry 3)
  - `ReserveStock` (Task → Lambda)
  - `Choose: StockReserved?` (Choice state, check output)
  - `ConfirmOrder` (Task)
  - `Catch` on `ReservePayment` → `VoidPayment` (Task)
  - `Catch` on any failure → `SendFailureNotification` via SNS.

Deploy with SAM. Test with Step Functions Local or stubbed Lambda responses.

### Lab 3: Event-driven architecture with EventBridge Pipes

Queue something: DynamoDB Stream from `Orders` table. When row updated with status="SHIPPED":

- EventBridge Pipe reads stream, filters by status.
- Target = SQS Queue for notifications.
- Separate SQS queue triggers Lambda with logic for user email.

Check lag in CloudWatch, ensure messages arrive within X seconds.

### Lab 4: JWT Authorizer with Cognito User Pool

1. Create Cognito User Pool with email sign-in.
2. Configure API Gateway with JWT Authorizer (Issuer + Audience).
3. Scope-based restriction: endpoint `/admin/**` requires `cognito:groups` claim contains "Admin".
4. Client demo: get with Postman, access token, call API.

### Lab 5: Serve "heavy" long-running batch with Express Workflows

Compare standard workflow vs express:

- Process 5000 records with annotations.
- Measure totalExecutionDuration and billable duration.
- Learn when to choose each.

---

## Entrevistas

1. **diference between HTTP API and REST API.**

   > HTTP is lower cost, lower feature set. For straightforward CRUD APIs open to public/lambda only, HTTP suficiente. REST si necesitas VTL, traditional usage plans, API keys management, request validation with models, or require direct integration with non-HTTP backend.

2. **cold starts — how predict and fix.**

   > Predict: CloudWatch `Init Duration`, high memory CPU-bound work, code bundling. Fix:捆包, SnapStart Java, provisioned concurrency for critical endpoint. NOTE: NOT for all functions — cost.

3. **design an image processing service that scales.**

   > API GW → SQS→Lambda with S3 put event. Deal with large files? S3 EventBridge direct event +Lambda hyperlink for download from pre-signed URL. Use SQS batching (10 messages) and error routing to DLQ. Include image metadata written to DynamoDB and start Step Fn post processing if multiple workers needed.

4. **cost comparison: Lambda vs EC2 vs Fargate for 24/7 API with 2M req/day.**

   > Depends on runtime. Node at 100ms 512MB avg: ~$30/mo. Same on EC2 c6i.large: $70/mo + LB $18/mo. Lambda cheaper for variable bursty traffic. Recommendation likely Revision: Start Lambda→Fargate+ASG when traffic is predictable.

5. **DynamoDB vs Aurora Serverless v2 for lambda-based e-commerce.**

   > DynamoDB: predictable low latency, no connection pool manage, HTTP for Lambda. Aurora Serverless: WHEN concurrent DB connections >10k (pooler issue), need relational queries impossible DDB (joinsanos complex), ACID multi-table transactions. Evaluate access patterns first (Module 05).

6. **Step Functions vs custom orchestrator.**

   > Built-in state management, retry handles, ASL declarative + visualization + metrics. Ideal when coordination > duration and complexity justifies declarative JSON overhead. Custom only for extreme throughput or proprietary simplicity.

7. **Cognito vs Auth0 vs build in-house.**

   > Cognito: cheapest, native AWS IAM integration for access to other resources via Identity Pools, trigger based customization with Lambdas. Auth0: superior Management API, free for smallMAU, better social/enterprise integrations without custom work. Build in-house only if regulatory needs exclude SaaS IAM.

8. **Architecture change: Monolito to Serverless trade-offs.**

   > Discuss Cold Start, connection pools to RDS, Observability, stateful session handling, long processes migration strategy, and cost variability (every request billed, not per-hour).

9. **exactly-once processing for payments via queues.**

   > Idempotency is key: at-least-once delivery from SQS, consumer-side dedup table, transaction isolated in single Lambda execution. Level: INTEGRACORS.

10. **Debugging distributed serverless application basics.**
    > X-Ray tracing activated, CloudWatch Logs Insights queries correlation, Payloads hidden by redaction policy, Observability Learn Lambda 2023.

---

## Checklist

- [ ] Entiendo differences entre HTTP API, REST API y ALB como entry point.
- [ ] Sé expliquan cold starts, cómo medirlos, y tres mitigaciones efectivas.
- [ ] Puedo elegir entre Provisioned vs Reserved Concurrency based on objective.
- [ ] He diseñado al menos una Saga orquestada con Step Functions y ASL JSON.
- [ ] Sé diferenciar SNS (fanout simple), SQS (trabajo), y EventBridge (routing+schema).
- [ ] Puedo integrar API Gateway con Cognito JWT Authorizer sin Lambda authorizer code.
- [ ] He implementado un producer/consumer idempotent pair with SQS and DynamoDB dedup.
- [ ] Entiendo límites de payload (API GW 10MB, SQS 256KB, Lambda 6MB direct sync).
- [ ] He implementado HTTPS redirects /authentication offload at CloudFront/API GW layer.
- [ ] He calculado cost estimation de un workload serverless previsualemente.

---

**Módulo 04 completo. La construcción de la infraestructura con Terraform/CloudFormation viene en el Módulo 06. Base de datos se profundiza en el Módulo 05.**
