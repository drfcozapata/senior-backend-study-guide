# Apéndice A — Mecanismos Suplementarios del Módulo 01

> Complemento al Módulo 01 de Arquitectura de Software Moderna. Aquí van los mecanismos técnicos detallados que se mencionaron pero no cabían en el flujo principal.

---

## 1. Fitness Functions (Funciones de Aptitud Arquitectónica)

Concepto de Neal Ford y Rebecca Parsons: *"Un fitness function es una métrica objetiva y automatizable que puede usarse para evaluar si un sistema cumple con las características arquitectónicas requeridas a lo largo del tiempo."*

### Tipos de Fitness Functions

| Tipo | Qué verifica | Ejemplo concreto |
|---|---|---|
| **ACID / Integridad de datos** | Transacciones no se pierden | Test de integración: escribe 100 transacciones, verifica que todas tienen idempotency key único |
| **Latencia** | p99 dentro del presupuesto | Lighthouse CI o k6 con thresholds: `http_req_duration` p(95) < 500 ms |
| **Acoplamiento** | No hay dependencias circulares ni imports ilegales | ArchUnit (Java), dependency-cruiser (Node), Warnings de compilación |
| **Seguridad** | No hay endpoints sin autenticación | OpenAPI spec validation que rechaza endpoints sin `security` tag, SAST scan en CI |
| **Escalabilidad** | Auto-scaling funciona bajo carga | Chaos engineering + load test: sube tráfico 10x, verifica que pods aumentan y latencia no se degrada más del 20% |
| **Observabilidad** | Todos los servicios emiten traces y métricas | Test que consulta CloudWatch / OTel collector y verifica que nuevos endpoints aparecen en dashboards |

### Implementación concreta (ejemplos)

#### a) Verificación de acoplamiento entre módulos (Dependency Fitness)

Para un **Modular Monolith** en TypeScript/NestJS:

```json
// .dependency-cruiser.js
module.exports = {
  forbidden: [
    {
      name: 'no-circular',
      severity: 'error',
      from: {},
      to: { circular: true }
    },
    {
      name: 'no-cross-module-deep-import',
      severity: 'error',
      from: { path: '^src/modules/orders' },
      to: { path: '^src/modules/payments/infrastructure' }
    }
  ]
}
```

Esto fuerza que dentro del módulo Orders solo puedas importar la API pública de Payments, nunca sus capas internas.

#### b) Verificación de latencia con k6 (Latency Fitness)

```javascript
// k6-latency-test.js
import http from 'k6/http';
import { sleep, check } from 'k6';
export const options = {
  thresholds: {
    http_req_duration: ['p(95)<500','p(99)<1000'],
    http_req_failed: ['rate<0.01']
  },
  stages: [
    { duration: '1m', target: 50 },
    { duration: '2m', target: 200 },
    { duration: '1m', target: 0 }
  ]
};
export default function () {
  const res = http.get('https://api.example.com/orders');
  check(res, {'status 200': r => r.status === 200});
  sleep(1);
}
```

Ejecutado en GitHub Actions cada noche o antes de un release candidate.

#### c) Fitness de observabilidad

Un test de integración que:

1. Llama un endpoint nuevo.
2. Verifica que en el trace backend hay `service.name` and `http.status_code`.
3. Verifica que CloudWatch tiene una métrica custom `orders_created_total` con valor incrementado.
4. Falla si faltan (significa: "has hecho cambios sin preparar observabilidad" = gate de CI/CD).

---

## 2. Contract Testing

Contract testing asegura que **una API (o evento) cumple su contrato** con los consumidores, sin ejecutar tests pesados de integración end-to-end.

### Por qué es crítico en arquitectura Senior

En un monolito es relativamente fácil hacer pruebas porque tienes el stack completo. En microservicios y serverless es imposible ejecutar toda la cadena cada commit. Contract testing cierra la brecha: verifica que Producer y Consumer todavía "hablan el mismo idioma sin necesidad de levantar el ambiente completo".

### Herramientas principales

| Herramienta | Lenguajes | Ideal para |
|---|---|---|
| **Pact** | JS, Java, .NET, Go, Python, Ruby | REST APIs, gRPC con extensiones |
| **Spring Cloud Contract** | JVM (Java/Kotlin) | REST, Messaging (Kafka) |
| **Schemathesis** | Cualquier HTTP API con OpenAPI | Fuzzing automático de contratos |
| **Buf / Protobuf lint** | gRPC | Linting de contratos + breaking change detection |
| **JSON Schema validators** | Cualquier | Validación simple de payloads de eventos |

### Ejemplo práctico: Consumer-Driven Contract con Pact (Node.js)

**Consumer side** (crea el contrato):

```javascript
// consumer.test.js
const { Pact } = require('@pact-foundation/pact');
const path = require('path');
const provider = new Pact({
  consumer: 'OrderService',
  provider: 'InventoryService',
  dir: path.resolve(__dirname, 'pacts'),
});

describe('Inventory Service Contract', () => {
  before(() => provider.setup());
  after(() => provider.finalize());

  it('reserves items for an order', async () => {
    await provider.addInteraction({
      uponReceiving: 'a request to reserve inventory',
      withRequest: {
        method: 'POST',
        path: '/api/v1/reservations',
        body: {
          orderId: 'order-123',
          items: [{ sku: 'pizza', quantity: 2 }]
        },
        headers: {'Content-Type': 'application/json'}
      },
      willRespondWith: {
        status: 201,
        body: { reservationId: 'res-abc', expiresAt: '2026-08-03T12:00:00Z' },
        headers: {'Content-Type': 'application/json'}
      }
    });
    // Consumer code calling mocked provider here...
    const result = await consumer.reserveItems({orderId: 'order-123', items: [{sku:'pizza', quantity: 2}]});
    expect(result.reservationId).toEqual('res-abc');
  });
});
```

El archivo Pact se publica al **Pact Broker**. Cuando el **Producer** hace cambios en el InventoryService, ejecuta los pacts contra su propio código (Verifier) para comprobar que **no rompe el contrato**.

### Schemathesis (Fuzzing automático de contratos API)

Si tienes una OpenAPI 3 spec (que todo API Senior debe tener), Schemathesis automatiza pruebas de contrato negativas:

```bash
schemathesis run openapi.yaml --base-url https://api.staging.example.com
```

Detecta automáticamente:

- Endpoints que devuelven status codes no definidos.
- Fields requeridos que el servidor devuelve null.
- Formatos equivocados (ej: fecha sin ISO8601).
- Queries que causan errores no tipados (SQL errors en lugar de 400 Bad Request).

### Event Contract Testing

Para eventos asíncronos (usados en Módulo 03), Pact también soporta "message pacts". Alternativa ligera: **esquemas JSON Schema en AWS Schema Registry** (para OpenAPI de eventos en EventBridge / Kafka). Cualquier cambio de esquema pasa primero por el Contract Test pipeline antes de romper producers y consumers.

---

## 3. Estrategias de Despliegue Internas

No son opcionales: el estilo de despliegue afecta la arquitectura misma. Un sistema que no puede desplegar dos versiones al mismo tiempo limita fuertemente su diseño (ej: no puedes tener contracts API independientes).

| Estrategia | Descripción | Ventajas | Peligros / Cuándo evitar |
|---|---|---|---|
| **Rolling Deployment** | Reemplaza instancias gradualmente. Mantiene la app running. | Simple, barato, bueno para stateless | Mezclas versiones por minutos. Riesgo si hay breaking change en DB o contratos |
| **Blue/Green** | Dos ambientes idénticos (blue = viejo, green = nuevo). Switch instantáneo después de validar green. | Instant rollback, downtime mínimo, valida en producción but solo para test users | Costo doble infra. Cambios de DB deben ser compatibles (expand/contract) |
| **Canary** | Despliega nuevo código a un pequeño % del tráfico. Analiza métricas y saca gradualmente. | Menor blastRadius. Gran control. Observabilidad fuerte | Requiere routing/measuring sofisticado (Istio, Flagger, AWS AppMesh). Confuso si métricas no están maduras |
| **Feature Flags** | Cambias comportamiento con toggle, no con despliegue. Separas deploy de release. | Reduce presión de rollback, habilita A/B, permite dark launches | Debt de flags, complejidad de lógica, necesidad de limpiar flags |
| **Shadow Traffic / Request Mirroring** | Envías copia del tráfico real a nueva versión sin responder al usuario. | Verifica comportamiento bajo carga real sin afectar usuarios | Dupla carga, precaución con datos sensibles, se necesita log/metrics strict |

### Elección arquitectónica

- **API crítica de pagos**: probablemente Blue/Green o Canaray con análisis automatizado de error rate.
- **Landing page o CMS**: Rolling es suficiente.
- **Cambio breaking en otra capa (ej: events version)**: expand & contract pattern (Primero deploy consumer que entienda nuevo esquema, luego producer emite nuevo esquema).
- **Serverless Lambda con aliases y weighted routing**: Built-in canary out-of-the-box (AWS CodeDeploy).

---

## 4. Plantilla completa de ADR (Architecture Decision Record)

Copia y pega. Ponlo en `docs/adr/NNN-slug.md` (ej `003-use-eventbridge-for-order-events.md`).

```markdown
# NNN. Title (short, problem focused)

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by NNN

## Context

What is forcing this decision? Constraints (legal, security, technical, team size, deadline)? What are the forces at play (scalability, maintainability, cost, speed)? Include facts from metrics, research, or existing issues.

## Decision

What was chosen? Keep it to the point. Be specific: "Use AWS EventBridge for domain events" not just "Use an event broker".

## Alternatives Considered

Why were the following not chosen (and what trade-off they would have)?

- Alternative A: Why considered, strengths, why rejected.
- Alternative B: same format.

Remember to evaluate at least two real options, not straw-man arguments.

## Consequences

### Positive
- Benefit 1 with quantification if possible ("reduce number of servers from 10 to 2").
- Benefit 2.

### Negative
- Cost or complexity added ("need to learn EventBridge semantics", "harder local testing").
- Risk of vendor lock-in.
- Migration plan if needed later.

## Plan

How will the decision be implemented? Milestones, estimations, who is responsible.

## Fitness Functions

How will we know it's still working?

- Metric 1 (e.g. "p95 processing latency < 300 ms").
- Metric 2 (e.g. "billing change endpoints respond 200").

## Review Date

When will we re-examine this? (May be quarterly, yearly, or when the trigger circumstance changes—traffic 10x, new regulation, team size x2).
```

### ADR contra "Ivory Tower"

Documentarlo en el repo hace que candidatos de entrevistas lo vean y entiendan que piensas de manera interdisciplinaria. Es un artefacto de arquitectura real, no documentación eterna e ilegible.

---

## 5. Comparativa SQL vs NoSQL aplicada a decisiones de arquitectura

Para Módulo 05 *Bases de Datos Distribuidas* entraremos en lo más profundo (DynamoDB internals, partition keys, hot keys). Aquí analizamos desde la óptica arquitectónica: **cómo cambia tu diseño si eliges SQL vs NoSQL**.

| Dimensión Arquitectónica | SQL / Relacional (PostgreSQL, MySQL, Aurora) | NoSQL (DynamoDB, Cassandra, MongoDB) |
|---|---|---|
| **Modelo de diseño** | Primero modelas el esquema (normalizado), luego codificas queries. retro-techreadme. Debe diseñarse para el futuro en gran medida. | Primero defines access patterns (Query PK=..., SK=...), luego defines las claves. "Schema on read", se diseña el storage para las consultas. Big shift. |
| **Consistencia** | ACID nativo, joins sofisticados, transacciones multi-tabla. | Eventual consistency la mayoría del tiempo. Applicative join (varias llamadas). Limited to single-item transactions in Dynamo. |
| **Escalabilidad** | Vertical + Read Replicas. Horizontal trick: Sharding manual (Vitess, Citus). | Horizontal by design basado en partition key. Auto-shards. High scale out of the box. |
| **Relaciones entre dominios** | Foreign keys explícitas. Unidos por SQL. | Desnormalización. Duplicación de datos intencional. Event-driven replication to bring data together. |
| **Cambio de esquema** | Migrations (ALTER TABLE). Dolor en producción alto. | Migrations son frecuentes pero menos estructurales: upsert new fields, backfill via stream events, rolling deployment. Menos sorpresas si tienes access pattern flexibility. |
| **Serverless friendliness** | Más complejo: connection pooling (RDS Proxy), cold start... no natural stateless handler. | DynamoDB does not require connections. SDK para queries. Cloud-native favorito. Serverless first-class citizen. |
| **Cost profile** | Payby uptime (CPU, connections) high base cost. | Pay per request or storage scales down to zero. High bursts can be expensive if not optimized. |
| **Testing** | Easy to spin up Docker Postgres locally to query. | Needs local DynamoDB (Docker image / DynamoDB Local) or mocked SDK. Adds testing complexity. |
| **Dominio recomendado** | Contabilidad, fundamentos complejos con ACID, reportes analíticos integrados (OLTP + ligero OLAP), alta integridad. | Catálogos producto, pedidos/tickets/events (alto volumen), sesiones/estado, IoT data streams, leaderboard/gaming. |

### Guidance: "SQL or NoSQL?" desde architectural standpoint

1. **Si necesitas transacciones complejas multi-recurso con foreign keys frecuentes** → SQL.
2. **Si necesitas high write throughput, escalabilidad geográfica, tu dominio tiene events, y eres serverless** → NoSQL más natural (dynamo con stream + lambdas).
3. **Modelo híbrido común**:
   - Order & Payment critical data en SQL.
   - Event log sourcing con DynamoDB streams + Lambda to publish to EventBridge.
   - Search/Read side projection en ElasticSearch from Kafka/Debezium.
4. **Datos técnicos pequeños como status**: DynamoDB great candidate for quick read/write (GSI query over entities by status).

La regla de oro no es preferencia personal, es **mapeo con el Bounded Context y sus patrones de acceso** (DDD-driven persistence).

---

## 6. Testing Piramidal aplicado a arquitectura

Mike Cohn introdujo la Pirámide de Testing: Many unit tests, fewer service/integration tests, very few end-to-end UI tests.

Porqué importa aquí: cada estilo arquitectónico tiene una **pirámide ideal distinta**. Un error frecuente es transferir la pirámide de un monolito a microservicios y enfrentar previsas y desilusiones ("our e2e suite is 2 hours long").

### Pirámide recomendada por tipo de arquitectura

| Arquitectura | Unit | Integration (DB/API) | Contract | E2E |
|---|---|---|---|---|
| **Monolito / Modular** | ~70% | ~20% | Raras (módulos internos generalmente dependen de shared-model) | ~10% |
| **Microservicios** | ~50% | ~20% | ~20-% | ~5-% (solo flujos críticos) |
| **Serverless / Event-Driven** | ~40-% | ~30-% (Localstack, DynamoDB local, SNS emulator) | ~20-% | ~10-% |
| **Legacy with no constraints** | ~30-% | ~50-% heavy because it's also regression safety net | Low | ~20-% |

### Ejemplos aplicados

- **Unit tests**: Fast, in-memory, mocks for ports. En Hexagonal Architecture son "domain logic + fake adapters".
- **Integration tests**: Capa de infraestructura. En AWS serverless probarías contra `dynamodb-local`, `sns-local` or real endpoints in a sandbox account, not every dev's machine.
- **Contract tests**: Como en sección anterior (Pact), asegura que Consumer y Provider están sincronizados.
- **End-to-End**: Smoke tests en producción. No intenten probar lógica profunda (eso es Integration + Unit). Intentan verificar que el sistema arranca y sirve el flujo básico principal (Happy Path).

### Anti-pattern clásico

Un cucumber Cypress suite gigante que desdice todo salvo el Registry. En sistemas serverless ese suite se degrada rápidamente ("flaky tests"), se ignora, y te quedas sin cobertura. La lección: enforce obesidad de convenios de testing en el pipeline CI/CD antes de mergear (coverage gate, e2e only critical flows).

---

## 7. Resumen rápido de todo el Apéndice

**Autoevaluation del Módulo 01 + apéndice**: Ahora no sólo sabes los nombres de los estilos. Sabes medirlos con Fitness Functions, validar contratos automáticamente, desplegar sin parar adolecer, crear un historial de decisiones con ADR, elegir entre SQL vs NoSQL por patrones de diseño, y ajustar la calidad de tests al producto y al team size. Ese es el paquete que distingue a un Senior Developer o a un Architect en las entrevistas de Amazon, Stripe o AgileEngine, y lo que evita los "vuelvo al punto inicial" production incidents during Fridays.

Reminder: los laboratorios del Módulo 0A1 existen disponibles para validar grado práctico. Si decides emprenderlos antes de ver Módulo 02 (Microservicios), te recomiendo fuertemente crear un mini Modular Monolith + al menos un ADR documentado y un Contract Test básico sobre un servicio HTTP interno.
