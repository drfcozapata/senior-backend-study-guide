# Módulo 06 — Infraestructura como Código (Terraform y CloudFormation)

> **Objetivo profesional:** Pasar de "crear recursos en la consola de AWS" a **tratar la infraestructura como producto de software**: reproducible, versionada, revisable, testeada y efímera. Aquí dominarás los dos enfoques dominantes — **Terraform** (declarativo, multi-cloud, estado explícito) y **CloudFormation** (nativo de AWS, sin estado propio, integrado con el ecosistema) — y sabrás cuándo elegir cada uno, cómo gestionar el estado, evitar el drift y estructurar módulos reutilizables.

> **Ubicación en la guía:** Se apoya en [Módulo 04](04-AWS-Serverless.md) (los recursos que vamos a desplegar: Lambda, API Gateway, DynamoDB, etc.) y [Módulo 05](05-Bases-de-datos-distribuidas.md). Conecta con los módulos de [Observabilidad](07-Observabilidad.md) y [Seguridad](08-Seguridad.md) (IAM), y con [CI/CD](12-CI-CD.md) (IaC dentro de pipelines). Conceptos base (CloudWatch, IAM) del [Glosario](00-Glosario.md).

---

## Introducción

Hacer `aws ec2 run-instances` o clicar en la consola "crear tabla" funciona _una vez_. Pero cuando tienes que reproducir el entorno de staging, replicar la prod en una nueva región, auditar quién cambió qué, o recuperar un ambiente destruido un viernes a las 5pm, el "click-click-click" es un **incidente esperando ocurrir**.

La **Infraestructura como Código (IaC)** es la práctica de **describir la infraestructura en archivos declarativos** que viven en control de versiones y se aplican de forma automática y repetible. No es solo "automatización": es aplicar a la infraestructura las mismas disciplinas del desarrollo de software — revisión de código, versionado, revisabilidad, testing, y ciclo de vida controlado.

**Beneficios reales (contextualizados):**

- **Reproducibilidad:** el mismo código produce el mismo ambiente (casi\* siempre — ver drift).
- **Velocidad:** de días/horas a minutos; entornos efímeros on-demand.
- **Revisabilidad y traceability:** cada cambio es un diff en PR, con autor y razón.
- **Manejo de ciclo de vida:** destruir y recrear entornos sin miedo ni acumulación de basura.
- **Multi-entorno consistente:** dev/stage/prod generadas desde la misma fuente con diferencias mínimas.

**La crítica honesta que un senior debe manejar:**

- IaC **no es gratis**: el estado y el drift son complejidades nuevas.
- El "código" falla por **desincronización** (drift) entre lo declarado y lo real.
- Requiere **disciplina operativa**: quién aplica, cuándo, con qué permisos, en qué orden.
- Herramientas distintas = curvas de aprendizaje y ecosistemas propios.

Elegir entre Terraform y CloudFormation no es solo "me gusta X": es una decisión de **estado, ecosistema, equipo y portabilidad** que veremos en detalle.

---

## Conceptos

### ¿Qué es exactamente IaC?

Es describir el **deseado final** (declarativo) o la **secuencia de pasos** (imperativo) para construir infra. Dominan los enfoques **declarativos** (Terraform, CloudFormation, Pulumi, AWS CDK): tú dices _qué quieres_, la herramienta calcula y aplica _cómo llegar_.

**Declarativo vs imperativo:**

- **Imperativo:** "crea una tabla, luego agrega una política, luego espera 10s". (Ansible, scripts). Vulnerable a no-idempotencia.
- **Declarativo:** "quiero que exista una tabla con esta clave y GSI". La herramienta hace el diff con el estado actual y solo cambia lo necesario.

### Los tres pilares de IaC reproducida

1. **Estado (desired state + actual state).** La herramienta guarda qué existe (el "state") y lo compara con lo declarado.
2. **Plan / Diff.** Antes de aplicar, muestra qué se creará/modificará/destruirá (la seguridad de `terraform plan` o el Change Set de CloudFormation).
3. **Aplicación idempotente.** Volver a aplicar el mismo código no debe producir cambios si ya está todo correcto.

### Infra como producto frente a scripts

Un script "de despliegue" que hace 15 CLI commands no es IaC robusta: no sabe _qué estado existe_, no remedia drift, no hace rollback atómico. IaC real se distingue por:

- Conocimiento del estado y capacidad de plan/diff.
- Módulos y reutilización.
- Ciclo de vida (crear/actualizar/destruir) gestionado.
- Integración con testing y CI/CD.

### Modelo de drivers: Terraform vs CloudFormation vs CDK/Pulumi

| Herramienta        | Modelo                                          | Estado                                                 | Ecosistema                                                   | Portabilidad   |
| ------------------ | ----------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------ | -------------- |
| **Terraform**      | HCL declarativo, provider multi-cloud           | Estado externo (local/S3+lock)                         | Multi-cloud (AWS, GCP, Azure, on-prem)                       | Alto           |
| **CloudFormation** | YAML/JSON declarativo nativo AWS                | **Sin estado propio** (la nube es la fuente de verdad) | Solo AWS, pero integrado con todo (SAM, stacks, Change Sets) | Bajo (lock-in) |
| **AWS CDK**        | Código (TS/Python/Go) que genera CloudFormation | Igual que CFN                                          | AWS+ (genera CloudFormation)                                 | Medio          |
| **Pulumi**         | Código (TS/Python/Go) con backends              | Estado externo                                         | Multi-cloud                                                  | Alto           |

> "AWS CDK" y "Pulumi" son de _programación_ pura, mientras Terraform/CFN son puramente _declarativos_. El que se aprende senior es **pensar en el modelo de estado y de errores**, más que en la sintaxis, porque eso es lo que se repite en cualquier herramienta.

### Terraform: piezas clave (recorrido técnico)

**Bloques elementales del HCL (HashiCorp Configuration Language):**

```hcl
# Provider: el conector con la nube (AWS)
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {}
}

variable "environment" {
  type    = string
  default = "dev"
}

resource "aws_dynamodb_table" "orders" {
  name         = "orders-${var.environment}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "PK"
  range_key    = "SK"

  attribute {
    name = "PK"
    type = "S"
  }
  attribute {
    name = "SK"
    type = "S"
  }
}

output "table_arn" {
  value = aws_dynamodb_table.orders.arn
}
```

**El flujo de trabajo del estado:**

1. **Terraform init** → descarga providers, configura el backend.
2. **Terraform plan** → lee el estado, calcula el diff desired vs actual, muestra acciones.
3. **Terraform apply** → ejecuta el plan y actualiza el estado.
4. **Terraform destroy** → destruye los recursos gestionados (y alerta del `prevent_destroy`).

### CloudFormation: el contrapunto de AWS

CloudFormation describe stacks (grupos de recursos) con YAML/JSON. La **nube misma es la fuente de verdad**: CFN consulta el estado real de los recursos; no guarda un archivo de estado aparte.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Stack de pedidos

Resources:
  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub 'orders-${AWS::StackName}'
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: PK
          AttributeType: S
        - AttributeName: SK
          AttributeType: S
      KeySchema:
        - AttributeName: PK
          KeyType: HASH
        - AttributeName: SK
          KeyType: RANGE

Outputs:
  OrdersTableArn:
    Value: !GetAtt OrdersTable.Arn
```

**Conceptos CFN clave:**

- **Stack:** unidad de rollback atómico; si algo falla, revierte.
- **Change Set:** equivalente a `terraform plan` (muestra los cambios antes de aplicarlos).
- **Drift detection:** compara el modelo deseado con el real (glitches por cambios manuales).
- **Nested stacks / StackSets:** reutilización y despliegue multi-cuenta.
- **SAM (Serverless Application Model):** extensión de CFN orientada a serverless (función, tabla, API Gateway) que reduce verbosidad.

---

## Arquitectura

### Topologías de gestión de entorno

**IaC — cómo orquestas múltiples entornos:**

```
┌──────────────────────────────┐
│ Código IaC en Git (branch)   │
│   main/ : prod infra         │
│   develop/ : stage infra     │
└──────────────────────────────┘
        │ (CI/CD pipeline: plan + apply)
        ▼
┌───────────────┬───────────────┬────────────────┐
│ Workspace dev │ Workspace stg │ Workspace prod │
│ backend s3:   │ backend s3:   │ backend s3:    │
│ bucket-iac-   │ bucket-iac-   │ bucket-iac-    │
│ dev/terraform │ stg/...       │ prod/...       │
└───────────────┴───────────────┴────────────────┘
```

Versión en diagrama (Mermaid):

```mermaid
flowchart TB
    subgraph GIT["Repositorio IaC (Git)"]
        MAIN["main → prod"]
        DEVBR["develop → stage"]
    end

    subgraph CI["CI/CD pipeline"]
        PLAN["terraform plan"]
        APPR["Aprobación manual"]
        APPLY["terraform apply"]
    end

    subgraph STATE["Backends de estado (S3 + lock DynamoDB)"]
        S1["workspace dev"]
        S2["workspace stg"]
        S3["workspace prod"]
    end

    subgraph AWS["AWS"]
        DEV[("dev cuenta/entorno")]
        STG[("stage cuenta/entorno")]
        PROD[("prod cuenta/entorno")]
    end

    MAIN --> CI
    DEVBR --> CI
    CI --> APPR
    APPR --> APPLY
    APPLY --> S1 & S2 & S3
    S1 --> DEV
    S2 --> STG
    S3 --> PROD

    classDef aws fill:#FF9900,stroke:#232F3E,color:#fff;
    class S1,S2,S3,DEV,STG,PROD aws;
```

- **Terraform workspaces:** aislan el estado por entorno dentro de un mismo código (ej. `dev`, `staging`, `prod`). Cada uno tiene su propio state en el backend.
- **StackSets / multi-account:** despliega el mismo stack en muchas cuentas (muy usado en el patrón de cuentas múltiples con AWS Control Tower / Landing Zone).

### Separación de responsabilidades/pipelines

**Patrón maduro de ambiente gestionado por IaC:**

1. **Infra base (network/VPC/IAM):** se crea una vez, poco cambiante, gobernada por un equipo platform.
2. **Infra de aplicación (Lambda, API, DynamoDB):** cambia con cada release; pipeline IaC aplica.
3. **Selector de entorno:** variables `environment`, `region`, `project` parametrizan todo.

Esto sigue la filosofía de **plataforma vs. aplicación** del [Módulo 02](02-Microservicios-y-DDD.md) (Team Topologies).

### Terraform Cloud / remote state

Un tema senior crítico: **el estado remoto y el bloqueo**. Con un equipo o en CI:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-iac-state"
    key            = "envs/prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"   # locking contra concurrencia
  }
}
```

- **S3** te da el almacenamiento central del state.
- **DynamoDB (lock table)** implementa el _locking_ → dos CI corriendo `apply` a la vez no se pisan.
- Alternativa: **Terraform Cloud** (estado + locking + remote run + plan en la nube, ideal para revisión de plan como PR).

---

## Internals

### Cómo funciona el state, el plan y el backend

**Flujo interno de Terraform (alto detalle):**

1. Lee configuración `.tf`.
2. Construye un **grafo de recursos** (dependencias implícitas por referencias: `aws_dynamodb_table.orders.arn`).
3. Carga el **state** desde el backend.
4. Hace el **diff** (graph traversal) → "Resource actions: 0 to add, 1 to change, 1 to destroy".
5. Aplica en **orden topológico** (respetando dependencias).
6. Persiste un **nuevo state** tras cada éxito.

**¿Por qué el estado importa tanto?**

- Mapea el **ID real** de cada recurso (`arn`, nombre) con el bloque del código.
- Si el state se pierde/corrompe y no está en remote, Terraform **no sabe** qué recursos creó → los huérfanos se quedan en la nube (y la factura).
- Por eso el **estado remoto con locking y backup** es obligatorio en equipos.

**Drift:**
Cuando alguien modifica un recurso a mano en la consola, el modelo deseado (código) deja de coincidir con el real. `terraform plan` lo detecta y propondrá revertir (o ignorar si configuraste `lifecycle { ignore_changes }`). CloudFormation lo expone vía _Drift Detection_.

### Manejo de errores y rollback

- **CloudFormation:** rollback automático **atómico** del stack si falla la creación/actualización. Con `DeletionPolicy` y `UpdateReplacePolicy` controlas qué pasa al reemplazar/eliminar un recurso (Retain/Snapshot/Delete).
- **Terraform:** **no** hace rollback automático. Si falla `apply`, deja aplicados los pasos que sí lograron (parcial). Por eso el **plan** y las opciones `-target` para aplicar en fases, y la disciplina CI que re-planee / vuelva a `apply` hasta validar.
- **Estrategia rollback IaC:** recreación desde el código (cuando los datos son efímeros) o `replace` controlado con `create_before_destroy`.

```hcl
resource "aws_instance" "app" {
  lifecycle {
    create_before_destroy = true  # crea la nueva antes de destruir la vieja
  }
  ...
}
```

### Prevención de destrucción accidental

```hcl
resource "aws_dynamodb_table" "critical" {
  lifecycle {
    prevent_destroy = true
  }
}
```

Protege recursos que no admite pérdida (bases con datos). Es una de las primeras "seguridad IaC" que un senior configura.

### Terraform vs CloudFormation: el trade-off en profundidad

| Dimensión                        | Terraform                                                                     | CloudFormation                                                         |
| -------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| **Estado**                       | Explícito, externo (S3/DynamoDB/Terraform Cloud) — control y riesgo de estado | Implícito — la nube como single source of truth, sin archivo de estado |
| **Rollback**                     | Manual (no automático) — creador del plan                                     | Automático atómico por stack                                           |
| **Lenguaje**                     | HCL declarativo + expresiones                                                 | YAML/JSON + funciones `!Ref`, `!GetAtt`, `!Sub`                        |
| **Multi-cloud / multi-provider** | Sí (nativo: AWS, GCP, Azure, on-prem)                                         | Solo AWS                                                               |
| **Reutilización**                | Módulos robustos + registroy                                                  | Nested stacks, StackSets, SAM                                          |
| **Drift**                        | Detectado en plan como diff que puedes revertir                               | Detección explícita de drift                                           |
| **Ecosistema/precedencia**       | Comunitario enorme de providers; estándar de facto multi-cloud                | Nativo AWS, integrado con IAM/roles/KMS/Tagging nativos, CDK lo genera |
| **Para who**                     | Equipos multi-cloud, o quienes quieren estado/reusabilidad/HCL                | Equipos 100% AWS que valoran nativo/serverless y rollback automático   |

**Regla senior rápida:**

- 100% AWS, priorizas serverless, quieres rollback atómico y menor overhead de estado → **CloudFormation/ SAM / CDK**.
- Multi-cloud, necesitas estado explícito, portabilidad, o team ya experto en HCL → **Terraform**.
- No es "mejor"; es **trade-off de estado/extensibilidad/ecosistema**. La respuesta correcta en entrevista es argumentar con el contexto del equipo y las necesidades.

---

## Patrones

### Módulos reutilizables (Terraform)

Un módulo es un directorio con `.tf` que encapsula un recurso/referencia con entradas/salidas claras. Patrón de equipo para reutilizar en varios entornos:

```hcl
# modules/table/main.tf
variable "table_name" { type = string }
variable "environment" { type = string }
resource "aws_dynamodb_table" "t" {
  name = "${var.environment}-${var.table_name}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key = "PK"
  range_key = "SK"
  # ... attribute definitions
}
output "table_arn" { value = aws_dynamodb_table.t.arn }
```

```hcl
# entornos/dev/main.tf
module "orders_table" {
  source      = "../../modules/table"
  table_name  = "orders"
  environment = "dev"
}
```

### Flight patterns: codigo de despliegue avanzado

Se detallan en el [Apéndice de Módulo 01](appends/01-Arquitectura-de-Software-Moderna-Apendice-A.md); en IaC los automatizas declaramos **(blue/green, canary)**:

- Estás **definiendo infra**, no solo funciones: por tanto las estrategias se codifican en el mismo IaC (ej: versiones de Lambda alias + weights).
- `create_before_destroy` permite la estrategia blue/green real a nivel de recurso.

### Sentinel / policy as code

Estándar para imponer reglas de infra (ej: "nadie puede crear una tabla sin encriptación en reposo", "no expongas buckets públicos"):

- **Terraform Sentinel** / **Checkov** / **tfsec** (open source security scanners de HCL).
- Se ejecuta en CI/apply y hace _fail-closed_ antes de tocar infra.

Ejemplo (tfsec ideal: la típica alerta):

```hcl
# aws_s3_bucket with default encryption S3_129
```

### State isolation por entorno

Para aislar `dev/stage/prod`:

- Backend S3 con key distinta por entorno.
- O **workspaces** por entorno + variables overhead.
- Mejor práctica moderna: **una carpeta por entorno** con su propio `backend` (separación total de state → un fallo en dev no toca prod).

### Paridad de entornos y el deployment pipeline (env parity + immutable)

IaC no termina en "creo la infraestructura": el objetivo senior es que **el entorno completo sea reproducible desde version control y se despliegue como si fuera otro artefacto del código** — la pieza que conecta IaC con CI/CD (Módulo 12). Tres prácticas que todo diseñador de pipelines debe conocer de memoria:

1. **Single source of truth para todo el sistema.** No solo el código de la app va a Git: también los scripts de packaging/deploy, las migraciones de esquema, los templates de cloud (CloudFormation/Terraform) y la configuración de la infra base. Así puedes **recrear todo el entorno de producción** a partir de lo que hay en version control, no solo el binario. (Suele haber órdenes de magnitud más opciones de configuración en el _entorno_ que en el _código_ — por eso el entorno debe estar en version control más que el propio código.)

2. **Entornos efímeros y "easier to rebuild than to repair" (infraestructura inmutable).** Si puedes recrear el entorno on-demand desde código, ante un fallo _reconstruyes_ (cattle) en lugar de _reparar_ a mano (pets). Prohibir cambios manuales en producción y **matar/reemplazar instancias** elimina la variación de config y el drift. El estado y el drift vistos arriba son la cara operativa exacta de esta inmutabilidad.

3. **Definición de "done" = corre en un entorno production-like.** El despliegue no se valida "en mi máquina": se considera _done_ solo lo que se **construye, despliega y confirma que corre** en un entorno lo más parecido a producción (carga y dataset production-like), e idealmente con las mismas herramientas de monitorización/despliegue que prod. Esto detecta el grueso de los fallos de deploy mucho antes del release y es lo que hace que el pipeline sea seguro.

> El _deployment pipeline_ es la línea completa de valor desde code check-in hasta producción: todo en version control, entorno creado y despliegue automatizados, ejecución on-demand. Es aquí, no en un botón manual, donde el plan de Terraform y el apply de CloudFormation se vuelven _seguros y frecuentes_. _(Fuente: The DevOps Handbook, cap. 9, "Create the Foundations of Our Deployment Pipeline"; reforzado en The Phoenix Project, caps. 30–31.)_

### Testing de infraestructura

- **Static:** tfsec/checkov + `terraform fmt` + `validate`.
- **Planek:** `terraform plan -detailed-exitcode` para verificar que no hay cambios no deseados.
- **Dynamic (Terratest):** Go test que `apply`, valida pre-provisioned, sometimes y destruye (integración real). Módulo 01 caso 2.

---

## Casos reales

### Caso 1: Migración de infra por clic a Terraform (fintech)

Una fintech tenía clustering manual de 40 microservices en la consola. Adoptaron Terraform con **estado remoto S3 + lock DynamoDB** y **workspaces por entorno**.

- **Resultado:** despliegues de infra que tomaban horas y requerían un "pico de conocimiento" ahora son PRs con plan diffs revisables; el equipos 4x más en deploy.
- **Lección:** el 90% del duro fue ordenar el **estado y el flujo (plan previo en CI)**. La migración a la nube del estado remotó fue el "acierto de seguridad" evitando perdida de rastreo tras semanas de trabajo.

### Caso 2: CloudFormation + SAM en start serverless puro

Startup serverless 100% AWS usa CloudFormation via SAM.

- **SAM** describe functions + API + DB en un único template, con `sam build` + `sam deploy`.
- **Resultado:** rollback atómico automático (Fallos revertir), integración con native IAM/KMS, tagging central. Dificultad baja en day-to-day.

### Caso 3: Control de costo/cambio con policy-as-code + template evaluaciones

Empresa grande con quince cuentas adopta **AWS Control Tower + Terraform multi-account**:

- Templates IaC generan el catálogo de servicios estándar.
- **Checkov en CI** rechaza cualquier PR que intente crear un bucket público o tabla sin encriptor.
- **Resultado:** el número de _security misconfigurations_ bajo 80% en dos trimestres; governance automatizada sin comité manual.

---

## Laboratorio

### Lab 1: Terraform de tu stack serverless

1. Crear `main.tf` que despliegue: DynamoDB orders (single-table), Lambda handler, API Gateway, y el EventBridge rule.
2. `terraform init`, `plan`, `apply` en `dev`.
3. `terraform show` e inspecciona el state y el ARN de salida.
4. Borrar con `terraform destroy` (preparado).

Requisito: Remote state en S3/DynamoDB para el _locking_.

### Lab 2: Estado, lock y concurrencia

1. Configura el **backend S3 + DynamoDB lock table**.
2. Lanza dos `terraform apply` simultáneos (ej: dos terminales). Verifica que el segundo **bloquea** hasta que el primero termina o `acquiring state lock` sale.
3. `terraform force-unlock <lock-id>` para simular un bloqueo huérfano.

### Lab 3: Drift

1. Crea un recurso (ej: una tabla) con Terraform.
2. En la consola AWS, **cámbialo a mano** (agrega un GSI o cambia billing).
3. `terraform plan` → observa que propone reconciliar el drift.
4. Configura `lifecycle { ignore_changes = [...] }` y repite plan.
5. Equivalente CFN: ejecuta **Drift Detection** sobre un stack y compara.

### Lab 4: Módulo reutilizable

Porta el ejemplo `modules/table` + dos entornos (dev/prod) usando la misma fuente. Cambia algo en prod; verifica que dev no se ve por el aislamiento de state.

### Lab 5: policy-as-code (tfsec/checkov) + CI

1. Instala tfsec/checkov y agrega reglas (bucket público, encriptación DynamoDB).
2. Haz que fallen reproses por no cumplir.
3. Configura un GitHub Action que ejecute `terraform fmt -check`, `validate`, `checkov` al hacer PR (enlaza a Modulo 12).

---

## Entrevistas

1. **Terraform vs CloudFormation: ¿cuál elijes y por qué?**

   **Orientación:** Argumenta _según contexto_, no por preferencia personal. Muestra que entiendes el trade-off de estado, ecosistema y rollback.

   **Respuesta de un senior:** "La elección depende del contexto, no de la moda. Para _multi-cloud_ o portabilidad, elijo _Terraform_: un solo lenguaje (HCL) y un modelo de estado explícito que no te ata a AWS, con un enorme ecosistema de providers. Para un equipo _100% AWS_ que usa _serverless_, recomiendo _CloudFormation_ (y por encima _SAM_/_CDK_): es nativo, el estado lo gestiona AWS, y tiene _rollback atómico por stack_ que Terraform no te da de fábrica. El trade-off real que señalo no es 'mi herramienta favorita': es qué gestión de estado y qué nivel de control quieres. Terraform te da control fino y estado manejable pero hay que cuidar el backend y el locking; CloudFormation te simplifica la vida dentro de AWS pero es verboso y menos portable. A veces uso ambos: CFN/SAM para serverless y Terraform para la capa multi-cloud."

2. **¿Qué es exactamente el estado de Terraform y cómo lo proteges?**

   **Orientación:** Debes explicar que el estado mapea recursos reales con los bloques, y la seguridad del backend remoto.

   **Respuesta de un senior:** "El estado de Terraform es el _mapa_ entre lo que declaré en HCL y los recursos reales que existen en la nube: guarda los IDs y atributos. Con él, `plan` sabe qué crear, cambiar o destruir. Como es el corazón del proceso, lo _protejo_: nunca lo guardo local ni lo comiteo, lo pongo en un _backend remoto_ —S3 con _versioning y encriptación_ más _locking_ vía DynamoDB o Terraform Cloud— y lo separo por entorno/workspace con `-backend-config`. Importante: si lo pierdes, dejarías recursos _huérfanos_ —existen pero Terraform ya no los conoce—, por eso backup y versioning son obligatorios. La regla senior: el estado es un artefacto sensible (contiene secretos y referencias), se trata como un recurso auditado y siempre remoto."

3. **¿Cómo manejas el bloqueo de estado y la concurrencia?**

   **Orientación:** Explica el lock (DynamoDB/Terraform Cloud), que bloquea y espera, y el force-unlock solo cuando es seguro.

   **Respuesta de un senior:** "El backend remoto gestiona un _lock_: cuando corre `apply`, se toma un bloqueo (en DynamoDB o en Terraform Cloud) para que dos applies simultáneos no corrompan el estado. Si otro proceso intenta aplicar, _bloquea y espera_ ('holding state lock') hasta que el primero libere, o falla si el lock está tomado. Es esencial porque dos writes concurrentes al estado romperían el mapeo. Si un proceso murió y dejó el lock _colgado_ (stale), hago `terraform force-unlock <id>` solo _después_ de verificar que ningún apply activo está corriendo, porque forzar con algo en ejecución podría sobrescribir estado. Mejor prevención: los pipelines centralizan la ejecución (CI + cola), así la concurrencia accidental es rara; el lock es la red de seguridad."

4. **¿Cómo detectas y solucionas el drift?**

   **Orientación:** Define drift y cómo detectarlo (plan/detect), distinguir cambios legítimos vs accidentales, y `ignore_changes`.

   **Respuesta de un senior:** "El _drift_ es cuando el estado real de la nube difiere de lo que el código declaró —alguien tocó el recurso a mano, en la consola o por otro tool. Lo detecto con `terraform plan`, que computa la desviación; en CloudFormation uso _Drift Detection_ por stack, que marca si los recursos del stack divergen de la plantilla. Para solucionarlo, distingo la causa: si el cambio es _legítimo_ (p. ej. algo que un job otra herramienta mantiene y no debe controlar Terraform), lo declaro con `ignore_changes` para que deje de marcarse como drift; si es _accidental_ o indeseado, `apply` revierte hacia el estado declarado. La clave senior es no borrar drift a ciegas: tengo un pipeline y _alarmas de drift_ para enterarme pronto, y policy para impedir que se `apply`e algo que no está aprobado."

5. **Diseña un flujo IaC para un equipo pequeño serverless.**

   **Orientación:** Buscan un repo por producto, backend remoto, entornos, plan en PR, apply en main con approval, policy-as-code y drift.

   **Respuesta de un senior:** "Para un equipo serverless pequeño, un flujo pragmático y seguro: un _repo por producto_ con el código y su infraestructura juntos. _Backend remoto_ (S3 + locking) separado por entorno. Uso _folders/workspaces_ por entorno (dev, stage, prod). El _CI_ corre `fmt`, `validate`, `plan` en cada _PR_ y comenta el plan para revisión; el _apply_ solo ocurre al fusionar a `main` y _con approval_ manual para entornos pre-prod/prod. Añado _policy-as-code_ (tfsec/checkov) para bloquear configuraciones inseguras antes del plan, y _alarmas de drift_ para detectar desviaciones. Todo _automatizado_: nadie aplica manualmente infraestructura fuera del pipeline. Es el punto justo entre velocidad (un solo repo) y seguridad (plan en PR, approval, policy) para no abrumar a un equipo chico con procesos pesados."

6. **Diferencias entre SAM y CloudFormation simple.**

   **Orientación:** Clarifica que SAM es un _abreviador serverless_ que genera CloudFormation.

   **Respuesta de un senior:** "SAM (Serverless Application Model) es una _capa de sintaxis_ sobre CloudFormation pensada para serverless: en lugar de definir verbosamente cada Lambda, API Gateway, DynamoDB y su función-IAM, escribe una _definición compacta_ (`Function`, `Api`, `Table`) y SAM _genera_ el CloudFormation equivalente al build. Te da conveniencias además: `sam local` para probar localmente, SimpleTable, transform que autocompleta la infraestructura. CloudFormation _complejo_ te da control total y explícito, pero es más verboso: defines letra a letra cada recurso, sus políticas y su IAM. Mi regla: si es serverless (funciones + API + eventos), SAM; si tengo infraestructura general (VPC, redes, EC2, multi-account) que no es solo funciones, CloudFormation plano. SAM _es_ CloudFormation por debajo, así que no compites, se complementan."

7. **¿Cómo haces el rollback en infraestructura?**

   **Orientación:** Contrasta el rollback automático de CloudFormation con el manual de Terraform (`plan/reapply`, `target`, `create_before_destroy`).

   **Respuesta de un senior:** "En _CloudFormation_ el rollback es casi automático: cada _stack_ es atómico, y si un deploy falla, CFN _revierte_ los cambios del stack a su estado anterior (o al stack anidado que corresponda). En _Terraform_ no hay rollback automático: la estrategia es re-ejecutar el _plan/reapply_ apuntando a la versión anterior del code, a veces con `-target` para acotar qué recurso toco primero. Para evitar idas y venidas uso _`create_before_destroy`_ (o _blue/green_): creo el recurso nuevo antes de borrar el viejo, de modo que si el nuevo falla, el anterior sigue vivo y puedo volver sin downtime real. En el pipeline, esto se combina con _feature toggles_: si un deploy introduce un bug, apago el toggle en vez de hacer un rollback costoso de infraestructura. La regla: minimizo la ventana de riesgo con blue/green y vuelvo rápido."

8. **¿Cómo evidencias que un cambio de infraestructura no rompe?**

   **Orientación:** Esperan plan en PR, Terratest, policy-as-code y detección de drift, todo en CI.

   **Respuesta de un senior:** "Montaría una cadena de verificación automatizada antes de tocar prod. (1) _plan en la PR_: el `terraform plan` se comentó para que el revisor vea el impacto exacto. (2) _Terratest_ (o testing de infra): un test que crea la infraestructura en un entorno efímero de verdad, verifica el comportamiento (p. ej. que la Lambda responde, que el bucket acepta escrituras) y la destruye, probando el estado real, no solo la sintaxis. (3) _policy-as-code_: tfsec/checkov bloquean reglas inseguras antes de planificar. (4) _detección de drift_: para detectar que la realidad se desvió del estado declarado. Todo corre en el _CI_ con gates: si el plan muestra cambios destructivos o el policy scanner falla, la PR no avanza. Así evidencia con datos que un cambio no rompe, no es una promesa."

9. **Protege una base con datos de la destrucción.**

   **Orientación:** Debes cubrir protección declarativa (prevent_destroy / DeletionPolicy Retain) + backups + confirmation manual.

   **Respuesta de un senior:** "Primero, protección a nivel de definición: en Terraform `prevent_destroy = true` en el recurso de base de datos, para que `terraform destroy` falle si intenta eliminarla; en CloudFormation, `DeletionPolicy: Retain` hace que, incluso si el stack se borra, el recurso y sus datos _permanezcan_. Añado _backups_ reales: snapshots/RDS automated backups o PITR en DynamoDB, ya que la protección declarativa evita borrarla, pero no un fallo de datos o un borrado manual fuera de IaC. Y en el _pipeline_ exijo _confirmación manual_ en prod: no basta el approve automático, una persona confirma antes de cualquier operación destructiva sobre datos. La triple capa —protección declarativa, backups, confirmación humana— es lo que hace que 'nunca se pierda la base' deje de ser un deseo y sea una garantía operativa."

10. **State isolation: workspaces vs folders.**

    **Orientación:** Explica cuándo conviene cada uno; folders = aislamiento total, workspaces = conveniencia con riesgo.

    **Respuesta de un senior:** "Ambos aíslan el estado, pero ofrecen distinto nivel de riesgo. Con _folders_ (una carpeta por entorno con su propio backend config), el estado de cada entorno está _totalmente separado_: lo que pasa en dev no toca prod, y el riesgo de error entre entornos es mínimo; es la opción que recomiendo para entornos _productivos_ y equipos que comparten infraestructura. Con _workspaces_ (una sola configuración con `terraform.workspace` diferenciando) es más _conveniente_: una sola codebase, y útil para entornos _ad hoc_ de desarrollo, pero introduce riesgo: al compartir código y ocultar el estado, un `apply` descuidado con el workspace equivocado puede afectar al stage/otros. Mi recomendación senior: _folders separados_ para dev/stage/prod (aislamiento real), y workspaces solo cuando quiero entornos temporales o experimentales donde el aislamiento estricto no es crítico. La separación es más importante que la comodidad."

---

## Checklist

- [ ] Diferencia claramente **declarativo vs imperativo** en IaC.
- [ ] Conozco los **tres pilares**: estado, plan/diff, aplicación idempotente.
- [ ] Entiendo cómo funciona el **backend/estado remoto** y por qué importa el **locking**.
- [ ] Sé configurar remote state **S3 + DynamoDB lock** y resolver bloqueos.
- [ ] Reconozco y manejo el **drift** (plan + ignore_changes / drift detection de CFN).
- [ ] Conozco los **bloques HCL básicos**: provider, resource, variable, output, backend, lifecycle.
- [ ] Domino conceptos CFN: **stack, Change Set, drift, nested/StackSet, SAM, DeletionPolicy**.
- [ ] Elijo entre **Terraform vs CloudFormation** según contexto con argumentos de estado/ecosistema.
- [ ] Uso **módulos reutilizables** y separación de entornos por folders/backend.
- [ ] Aplico **policy-as-code** (tfsec/checkov) y testing (fmt/validate/plan/Terratest).
- [ ] Protejo recursos críticos con `prevent_destroy` / `DeletionPolicy: Retain`.
- [ ] Integro IaC en **CI/CD** con plan en PR y apply controlado en main.

---

## Referencias y lecturas recomendadas

- **HashiCorp — Terraform documentation & AWS Provider** — https://registry.terraform.io/providers/hashicorp/aws/latest/docs (referencia canónica).
- **HashiCorp — Terraform Language (backends, state, lifecycle)** — https://developer.hashicorp.com/terraform/language
- **AWS — CloudFormation User Guide** — https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/
- **AWS — Change Sets** — https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html
- **AWS — Detect drift on a stack** — https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html
- **AWS — SAM (Serverless Application Model)** — https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/
- **"Terraform Up & Running"** — Yevgeniy Brikman (O'Reilly, 3rd ed.). El mejor libro práctico de Terraform.
- **"The Terraform Book"** — James Turnbull (LeanPub). Alternativa sólida.
- **tfsec** (seguridad en HCL) — https://github.com/aquasecurity/tfsec y **Checkov** — https://www.checkov.io/ (policy-as-code).
- **Terratest** — https://github.com/gruntwork-io/terratest (testing de infra con Go).
- **"The DevOps Handbook"** — Kim, Humble, Debois, Willis, Forsgren. Cap. 9 (deployment pipeline, env parity, infraestructura inmutable).
- **"The Phoenix Project"** — Gene Kim, Kevin Behr, George Spafford. Caps. 30–31 (la paridad de entornos y el objetivo de deploys frecuentes).

---

> _Más profundidad, diagramas y patrones avanzados en el [**Apéndice A**](appends/06-Infraestructura-como-Codigo-Apendice-A.md). Los módulos [**07 - Observabilidad**](07-Observabilidad.md) y [**12 - CI/CD**](12-CI-CD.md) continúan la automatización del entorno._
