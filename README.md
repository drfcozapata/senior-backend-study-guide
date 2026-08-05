# Senior Backend Study Guide

Guía de estudio a nivel **Senior** para diseño de software en sistemas distribuidos sobre AWS. Escrita como un libro técnico, orientada a preparar a un ingeniero para entrevistas y trabajo real en empresas del nivel de Amazon, Stripe, Airbnb, Netflix, Mercado Libre o AgileEngine.

> No es una guía para aprobar una certificación. Es una guía para **entender cómo se diseña software**.

---

## Cómo usar esta guía

- Cada módulo es un archivo Markdown independiente y autocontenido.
- El módulo 00 es un glosario vivo que crece con cada módulo: cuando encuentres un término que no domines, búscalo ahí.
- Los módulos se referencian entre sí; puedes seguir el orden numérico o saltar a lo que necesites usando el índice.
- La estructura interna de cada módulo (01–14) es siempre la misma: **Introducción → Conceptos → Arquitectura → Internals → Patrones → Casos reales → Laboratorio → Entrevistas → Checklist**. Los módulos 15 y 16 ajustan la forma a su contenido (banco de preguntas / roadmap).

---

## Índice de módulos

| #   | Módulo                                                                                                                                                 | Temas centrales                                                                                                                                      |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| 00  | [Glosario](00-Glosario.md)                                                                                                                             | Definiciones de todos los conceptos de la guía                                                                                                       |
| 01  | [Arquitectura de Software Moderna](01-Arquitectura-de-Software-Moderna.md) (+ [Apéndice A](appends/01-Arquitectura-de-Software-Moderna-Apendice-A.md)) | Clean, Hexagonal, Onion, Vertical Slice, SOLID, Twelve-Factor                                                                                        |
| 02  | [Microservicios y DDD](02-Microservicios-y-DDD.md) (+ [Apéndice A](appends/02-Microservicios-y-DDD-Apendice-A.md))                                     | Bounded Contexts, Aggregates, descomposición, Service Discovery                                                                                      |
| 03  | [Event-Driven Architecture](03-Event-Driven-Architecture.md)                                                                                           | Mensajería, Event Sourcing, CQRS, Saga, Outbox, delivery semantics                                                                                   |
| 04  | [AWS Serverless](04-AWS-Serverless.md)                                                                                                                 | Lambda, API Gateway, EventBridge, Step Functions, Cognito                                                                                            |
| 05  | [Bases de datos distribuidas](05-Bases-de-datos-distribuidas.md)                                                                                       | DynamoDB, patrones NoSQL, particionamiento, GSI/LSI                                                                                                  |
| 06  | [Infraestructura como Código](06-Infraestructura-como-Codigo.md) (+ [Apéndice A](appends/06-Infraestructura-como-Codigo-Apendice-A.md))                | Terraform, CloudFormation                                                                                                                            |
| 07  | [Observabilidad](07-Observabilidad.md) (+ [Apéndice A](appends/07-Observabilidad-Apendice-A.md))                                                       | OpenTelemetry, Prometheus, Distributed Tracing, RUM, alerting SRE                                                                                    |
| 08  | [Seguridad](08-Seguridad.md) (+ [Apéndice A](appends/08-Seguridad-Apendice-A.md))                                                                      | OAuth 2.1, JWT, OpenID Connect, IAM, mejores prácticas                                                                                               |
| 09  | [Diseño de APIs](09-Diseño-de-APIs.md) (+ [Apéndice A](appends/09-Diseño-de-APIs-Apendice-A.md))                                                       | REST, GraphQL, gRPC, Webhooks, versionado, idempotencia                                                                                              |
| 10  | [Escalabilidad y Resiliencia](10-Escalabilidad-y-Resiliencia.md) (+ [Apéndice A](appends/10-Escalabilidad-y-Resiliencia-Apendice-A.md))                | Caching, Sharding, Replication, Circuit Breaker, Backpressure                                                                                        |
| 11  | [Patrones de Diseño](11-Patrones-de-Diseño.md) (+ [Apéndice A](appends/11-Patrones-de-Diseño-Apendice-A.md))                                           | GoF, patrones de arquitectura, estilos                                                                                                               |
| 12  | [CI/CD moderno](12-CI-CD-Moderno.md) (+ [Apéndice A](appends/12-CI-CD-Moderno-Apendice-A.md))                                                          | GitHub Actions, estrategias de despliegue                                                                                                            |
| 13  | [AI-Assisted Development](13-AI-Assisted-Development.md) (+ [Apéndice A](appends/13-AI-Assisted-Development-Apendice-A.md))                            | Copilot, Claude Code, Cursor, Codex, OpenCode, agentes, MCP, context engineering                                                                     |
| 14  | [System Design Interview](14-System-Design-Interview.md) (+ [Apéndice A](appends/14-System-Design-Interview-Apendice-A.md))                            | Pensar como arquitecto, framework de entrevista, estimaciones, casos: Twitter/X, Netflix, Uber, WhatsApp, pagos, notificaciones                      |
| 15  | [Preguntas y ejercicios](15-Preguntas-y-Ejercicios.md) (+ [Apéndice A](appends/15-Preguntas-y-Ejercicios-Apendice-A.md))                               | Banco de preguntas por nivel (Junior/Mid/Senior) respondidas como senior, rubric 2026, ejercicios prácticos, debugging rounds y diseño de bajo nivel |
| 16  | [Roadmap y Proyecto Final](16-Roadmap-y-Proyecto-Final.md) (+ [Apéndice A](appends/16-Roadmap-y-Proyecto-Final-Apendice-A.md))                         | Roadmap de 10 semanas, capstone end-to-end (order & fulfillment), laboratorios integradores y laboratorio de ingeniería agencial                     |

---

## Conceptos transversales

Estos y otros conceptos aparecen repetidamente a lo largo de la guía y están definidos en el [Glosario](00-Glosario.md):

- **Teoría de sistemas distribuidos:** CAP Theorem, PACELC, ACID vs BASE, consistencia eventual
- **Semántica de entrega:** At-most-once, At-least-once, Exactly-once, idempotencia
- **Patrones de arquitectura:** Event Sourcing, CQRS, Saga, Outbox, Clean/Hexagonal/Onion/Vertical Slice
- **Patrones de resiliencia:** Retry, Circuit Breaker, Rate Limiting, Backpressure
- **Patrones de datos:** Cache-Aside, Read/Write-Through, Write-Behind, Sharding, Replication
- **Consenso y coordinación:** Leader Election, Leader/Follower, Consistent Hashing

---

## Estructura de directorios

```
senior-backend-study-guide/
│
├── README.md                    ← Este archivo (índice)
├── 00-Glosario.md               ← Glosario vivo de conceptos
├── 01-…16-*.md                  ← Módulos de estudio
│
└── appends/                     ← Apéndices (A) de cada módulo
    └── NN-…-Apendice-A.md
```
