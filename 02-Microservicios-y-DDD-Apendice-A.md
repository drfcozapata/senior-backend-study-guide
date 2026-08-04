# Apéndice A — Mecanismos Suplementarios del Módulo 02

> Profundización avanzada en mensajería, agregados, testing distribuido y técnicas operativas. Complementa a `02-Microservicios-y-DDD.md`.

---

## 1. Dirección de mensajes: Comandos, Eventos y Consultas

| Tipo | Naturaleza | Destinatario | Ejemplo |
|---|---|---|---|
| **Command** | Intención de cambiar estado (acción) | Un receptor específico; espera confirmación | `CreateOrder`, `CancelReservation`, `IssueRefund` |
| **Event** | Hecho ya ocurrido (declarativo) | Broadcast, múltiples suscriptores, sin dueño de respuesta | `OrderCreated`, `PaymentProcessed`, `UserRegistered` |
| **Query** | Solicitud de datos sin cambiar estado | Respuesta síncrona generalmente | `GetOrderById` |

**Anti-patrones:**
- Usar un **evento** esperando que alguien ejecute una acción: eso es Command.
- Construir un flujo de trabajo a base de resultados de Query sin tracking de estado.

---

## 2. Fan-in / Fan-out

### Fan-Out
Un mensaje dispara múltiples consumidores independientes.
```text
SNS Topic -> SQS Queue A (notifications)
          -> SQS Queue B (audit)
          -> Lambda (data enrichment)
```
**Riesgo:** un suscriptor lento atasca; filtramos los mensajes antes de encolar.

### Fan-In
Muchos productores → un procesador.
- Multiples producers -> SQS FIFO con `MessageGroupId`, o Kinesis.
- **Reto:** preservar el orden y deduplicar.

---

## 3. Request-Reply sobre canales asíncronos

Para tener semántica de respuesta usando colas:
- Cola temporal por request.
- **Correlation ID** en el mensaje resposta.
- Timeout de espera.
**Advertencia:** latencia alta; solo para budgets holgados (no para trading de mega consumo).

---

## 4. Agregados avanzados

### Bloated Aggregates
Un aggregate no debe ser un God object. Regla: posee solo entidades cuyo **ciclo de vida** depende estrictamente de él.
- **Violación:** `InvoiceAggregate` con `List<PaymentHistory>` si los pagos siguen tras archivar la factura.
- **Solución:** separar `InvoiceAggregate` (estado actual) y `PaymentHistoryAggregate` (ledger histórico), vinculados por UUID débil o query projection.

### Snapshots en Event Sourcing
Evitar replay de millones de eventos:
- Cada N eventos se guarda un snapshot (estado serializado en versión N).
- Restauración: leer snapshot + replay eventos posteriores.
- **Trade-off:** consistencia de snapshot si se cae durante la escritura (boundary transaccional).

### Domain Services vs Application Services (recuerdo Clean)
- **Domain Service:** reglas de negocio, estados. Puro.
- **Application Service:** coordinación técnica (transacciones, repositorios, seguridad, DTOs) — sin reglas de negocio.
- **Infrastructure:** drivers I/O (DB, HTTP, logging).

---

## 5. Contract Testing en profundidad

Ver Patrones del módulo principal a nivel Módulo 03/12, aquí la mecánica.

### Pact Bi-Directional
Para validación de esquemas sin ejecutar código el activo:
1. Provider sube OpenAPI spec al Broker.
2. Consumer valida compatibilidad contra el schema.
3. Cambio breaking → falla el pipeline automáticamente.

### Message contracts (EventBridge/Kafka)
Usar **"Pact message providers"** o **JSON Schema Registry** para validar que el evento emitido cumple el esquema esperado antes de activar reglas de enrutado, campañas hacerbreaking changes.

---

## 6. Gestión de Configuración a escala

La configuración es "la DB global no versionada" de los distribuitos.

### Patrones de external config
1. **Polling:** los servicios consultan interízos a AppConfig (AWS).
2. **Push/WebSocket:** refresh inmediato.
3. **Sidecar injection:** Envoy monta config como archivo; el servicio detecta el cambio.

**Seguridad:** la configuración también necesita control de acceso (quién puede cambiar retry policy / feature flags).

### Feature Flags
- **Corto lifetime.**
- **Default off** para features nuevos.
- **Granular targeting:** primero staff, luego beta, luego 100%.
- Con la APM para monitorear error rate al activar flag.

---

## 7. Service Mesh: ¿cuándo introducirlo? (nivel Senior)

**Signos de madurez:**
- >10 servicios y la gestión de sidecars/certificados/rutas te desborda.
- Necesides de traffic splitting L7 (canary).
- Observabilidad unificada independiente del idioma/framework.

**Costos honestos:**
- Overhead initial alto.
- Sidecars consumen CPU/RAM (latencia de proxy).
- Troubleshooting en dos capas.

**Recomendación:** empieza con managed (API Gateway + Cloud Map + Lambda). Solo evalúe mesh si la complejidad polyglot hace inalcanzable la consistencia operativa —entrenlean.

---

## 8. Disaster Recovery por servicio

### RPO / RTO
- **RPO (¿cuánta pérdida aceptamos?):** ledger financiero → ~0 (replicación región); sesiones → minutos.
- **RTO (¿cuánto fuera de servicio?):** UI crítico → <5min failover; reporting por lotes → horas.

Arquitectura: para RPO casi 0 no basta consistencia eventual; con sync redundos (thread que increment equal) o event sourcing con backups fiables.

---

## 9. Ejercicios de escenarios complejos

### Escenario 1: Recuperación de orquestación de pagos
`Step1: cargar tarjeta` ok, `Step2: reservar inventario` falla.
Compensación: `VoidCardTransaction` iniciado por Saga al recibir `StockReservedFailed`.
**Complicación:** ¿qué si la API de void está temporalmente caída? → Delayed retry → DLQ → intervención manual.

### Escenario 2: Hotspot de cache de catálogo
Producto viral golpea cache.
Solución: **read-through nahoy + background refresh** + rate limits edge.

### Escenario 3: Nightmare de dependencias
10 servicios Java8, 20 Java17. Un change de shared library rompe contrato.
- Mitigar: **semantic versioning** strict, **compatibility test grid** en CI.

---

## 10. Glosario extendido del Módulo 02

(para incorporar al Glosario principal en la fase de coherencia)

- **Domain Event:** registro de que algo pasó en el dominio.
- **Bounded Context:** frontera de consistencia semántica.
- **Aggregate Root:** punto de entrada para modificar el grafo del aggregate.
- **Saga:** secuencia de transacciones locales ligadas por eventos/compensaciones.
- **Outbox:** técnica para escribir el evento en la misma DB transaccional.
- **Strangler Fig:** reemplazo gradual del legacy.
- **Anti-Corruption Layer (ACL):** traducción entre modelos.
- **Bulkhead:** aislamiento de recursos para evitar fallo en cascada.
- **Circuit Breaker:** fail-fast ante fallos remote.
- **Retry policy:** reglas para fallos transitorios.
- **mTLS:** autenticación bidireccional por certificados.
- **Zero Trust:** postura de seguridad que asume breach.
- **Service Mesh:** capa de infraestructura para comunicación servicio-a-servicio.
- **Correlation ID:** etiqueta única para unir logs distribuidos.
- **SLI/SLO/SLA:** indicadores, objetivos, acuerdos.