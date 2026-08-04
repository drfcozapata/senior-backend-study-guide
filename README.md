# Senior Backend Study Guide

Guía de estudio a nivel **Senior** para diseño de software en sistemas distribuidos sobre AWS. Escrita como un libro técnico, orientada a preparar a un ingeniero para entrevistas y trabajo real en empresas del nivel de Amazon, Stripe, Airbnb, Netflix, Mercado Libre o AgileEngine.

> No es una guía para aprobar una certificación. Es una guía para **entender cómo se diseña software**.

---

## Cómo usar esta guía

- Cada módulo es un archivo Markdown independiente y autocontenido.
- El módulo 00 es un glosario vivo que crece con cada módulo: cuando encuentres un término que no domines, búscalo ahí.
- Los módulos se referencian entre sí; puedes seguir el orden numérico o saltar a lo que necesites usando el índice.
- La estructura interna de cada módulo es siempre la misma: **Introducción → Conceptos → Arquitectura → Internals → Patrones → Casos reales → Laboratorio → Entrevistas → Checklist**.

---

## Índice de módulos

| # | Módulo | Temas centrales |
|---|--------|-----------------|
| 00 | [Glosario](00-Glosario.md) | Definiciones de todos los conceptos de la guía |
| 01 | [Arquitectura de Software Moderna](01-Arquitectura-de-Software-Moderna.md) (+ [Apéndice A](01-Arquitectura-de-Software-Moderna-Apendice-A.md)) | Clean, Hexagonal, Onion, Vertical Slice, SOLID, Twelve-Factor |
| 02 | [Microservicios y DDD](02-Microservicios-y-DDD.md) (+ [Apéndice A](02-Microservicios-y-DDD-Apendice-A.md)) | Bounded Contexts, Aggregates, descomposición, Service Discovery |
| 03 | [Event-Driven Architecture](03-Event-Driven-Architecture.md) | Mensajería, Event Sourcing, CQRS, Saga, Outbox, delivery semantics |
| 04 | [AWS Serverless](04-AWS-Serverless.md) | Lambda, API Gateway, EventBridge, Step Functions, Cognito |
| 05 | [Bases de datos distribuidas](05-Bases-de-datos-distribuidas.md) | DynamoDB, patrones NoSQL, particionamiento, GSI/LSI |
| 06 | [Infraestructura como Código](06-Infraestructura-como-Codigo.md) (+ [Apéndice A](06-Infraestructura-como-Codigo-Apendice-A.md)) | Terraform, CloudFormation |
| 07 | [Observabilidad](07-Observabilidad.md) (+ [Apéndice A](07-Observabilidad-Apendice-A.md)) | OpenTelemetry, Prometheus, Distributed Tracing, RUM, alerting SRE |
| 08 | [Seguridad](08-Seguridad.md) (+ [Apéndice A](08-Seguridad-Apendice-A.md)) | OAuth 2.1, JWT, OpenID Connect, IAM, mejores prácticas |
| 09 | [Diseño de APIs](09-Diseño-de-APIs.md) (+ [Apéndice A](09-Diseño-de-APIs-Apendice-A.md)) | REST, GraphQL, gRPC, Webhooks, versionado, idempotencia |
| 10 | Escalabilidad y Resiliencia | Caching, Sharding, Replication, Circuit Breaker, Backpressure |
| 11 | Patrones de Diseño | GoF y patrones de arquitectura empresarial |
| 12 | CI/CD moderno | GitHub Actions, estrategias de despliegue |
| 13 | AI-Assisted Development | Copilot, Claude Code, Cursor, agentes |
| 14 | System Design Interview | Framework de entrevistas de diseño a nivel Senior |
| 15 | Preguntas y ejercicios | Banco de preguntas técnicas y ejercicios prácticos |
| 16 | Roadmap y Proyecto Final | Plan de estudio, laboratorios integradores, proyecto capstone |

*(Los enlaces se activan a medida que se genera cada módulo.)*

---

## Conceptos transversales

Estos conceptos aparecen repetidamente a lo largo de la guía y están definidos en el [Glosario](00-Glosario.md):

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
├── diagrams/                    ← Diagramas Mermaid (.mmd)
└── assets/                      ← Recursos complementarios
```
