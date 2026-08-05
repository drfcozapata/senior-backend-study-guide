# Apéndice A — Módulo 12: Implementaciones de referencia

> Este apéndice complementa [12-CI-CD-Moderno.md](12-CI-CD-Moderno.md) con implementaciones concretas en GitHub Actions: un pipeline CI completo (build once + artifacts + matrix + cache), OIDC keyless a AWS, entornos con protection rules, canary con telemetría, feature flag, GitOps con ArgoCD/Flux y release automation. Cada ejemplo prioriza *el patrón que resuelve* sobre la sintaxis.

---

## 1. CI completo: build una sola vez + artifacts (GitHub Actions)

El patrón *build once, deploy many*: el artifact se genera en CI y los jobs de deploy lo descargan, nunca re-build.

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read            # least-privilege GITHUB_TOKEN

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20, 22]   # matriz de versiones (testing farm moderno)
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm         # dependencias cacheadas (solo velocidad)
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      # El artifact es la fuente de verdad para el deploy:
      - uses: actions/upload-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
          retention-days: 30
```

**Punto senior:** los `needs: test` fuerzan que el build solo ocurra con tests verdes; el artifact lleva el SHA, así el deploy sabe *exactamente* qué binario es.

---

## 2. Entornos con protection rules + secretos por environment

Producción exige aprobación humana y no puede venir de cualquier branch. Los secretos de prod solo se exponen al job que referencia el environment (tras aprobar).

```yaml
# .github/workflows/cd.yml
name: CD
on:
  workflow_dispatch:
  push:
    tags: ["v*.*.*"]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:            # sin gates: corre automático
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
      - name: Deploy to staging
        env:
          API_URL: ${{ vars.STAGING_URL }}      # var del environment
        run: ./scripts/deploy.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:            # gates: required reviewers + wait timer + branch policy
      name: production
      url: https://example.com
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
      - name: Deploy to production
        env:
          DEPLOY_TOKEN: ${{ secrets.PROD_DEPLOY_TOKEN }}   # solo tras aprobar
        run: ./scripts/deploy.sh production

      - name: Smoke test post-deploy
        run: curl -fsS https://example.com/health || exit 1
```

**En Settings → Environments → production:**
- Required reviewers: `@release-team` (con *prevent self-review* activo).
- Wait timer: 5 minutes.
- Deployment branches: `main`, `release/*`, `v*.*.*`.

---

## 3. OIDC keyless a AWS (sin access keys)

El patrón de oro del cap. 8 y 12: GitHub emite un JWT firmado; AWS lo valida contra un role con trust policy acotada.

```yaml
# .github/workflows/deploy-aws.yml
name: Deploy to AWS
on:
  workflow_dispatch:

permissions:
  id-token: write            # necesario para emitir el token OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-oidc-prod
          aws-region: us-east-1
          role-session-name: github-actions
      - name: Verify identity
        run: aws sts get-caller-identity
```

Trust policy del role (en AWS):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:mi-org/mi-app:environment:production"
      }
    }
  }]
}
```

**Punto senior:** el `sub` acota a *repo + environment específico* — un PR de un fork no puede asumir el role de producción.

---

## 4. Canary con telemetría (mini cluster immune system)

Despliega a un 5%, mide una métrica, y **rechaza el rollout** si se desvía (la idea del *cluster immune system* del libro).

```yaml
# .github/workflows/canary.yml
jobs:
  canary:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Deploy 5% canary
        run: ./scripts/canary.sh start --percent 5 --tag "${{ github.sha }}"

      - name: Watch telemetry (p99 latency & error rate)
        id: watch
        timeout-minutes: 15
        run: |
          # espera a que la métrica se estabilice y decide
          result=$(./scripts/check_slo.sh --p99-threshold 250ms --error-threshold 0.01)
          echo "slo_ok=$result" >> "$GITHUB_OUTPUT"
          [ "$result" = "false" ] && echo "::error::SLO breached, aborting rollout"

      - name: Promote to 100% (solo si SLO ok)
        if: steps.watch.outputs.slo_ok == 'true'
        run: ./scripts/canary.sh promote --tag "${{ github.sha }}"

      - name: Rollback (si SLO roto)
        if: steps.watch.outputs.slo_ok != 'true'
        run: ./scripts/canary.sh rollback
```

---

## 5. Feature flag (rollback sin redeploy)

Release por switch, no por deploy. Ejemplo mínimo de SDK:

```ts
// flags.ts (ej: LaunchDarkly, Split, o flag service propio)
import { FeatureClient } from "@flags/sdk";

const flags = new FeatureClient({ envKey: process.env.FLAGS_KEY });

export async function isDarkLaunchEnabled(userId: string) {
  // 1% de usuarios mientras validamos, luego 5%, luego todos
  return flags.percentageRollout("new_checkout", 5, userId);
}

// Uso en el handler: la feature vive en prod pero invisible hasta encenderla
async function handleCheckout(req, res) {
  if (await isDarkLaunchEnabled(req.user.id)) {
    return newCheckoutFlow(req);   // dark launch: código en prod, off por defecto
  }
  return legacyCheckoutFlow(req);
}
```

**Rollback por flag:** `flags.set("new_checkout", false)` — sin deploy. **Riesgo (flag debt):** limpiar el flag cuando llegue al 100%; testear con flags *on*.

---

## 6. GitOps con Flux (o ArgoCD): declarativo, pull, reconciliación

El CI no toca el clúster: solo actualiza el manifest en Git; el operador pull y reconcilia.

```bash
# 1. Bootstrap del cluster desde Git (estado deseado = repo GitOps)
flux bootstrap github \
  --owner=mi-org \
  --repository=mi-gitops-repo \
  --branch=main \
  --path=./clusters/prod

# 2. Definir una app que se reconcilia desde el repo de la app
cat > clusters/prod/apps/mi-app.yaml <<'EOF'
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: mi-app
  namespace: flux-system
spec:
  interval: 1m                    # reconciliación continua (drift auto-correct)
  path: ./apps/mi-app/overlays/prod
  prune: true
  sourceRef:
    kind: GitRepository
    name: mi-app-repo
  postBuild:
    substituteFrom:
      - kind: Secret              # SOPS / External Secrets: nunca secretos en claro
        name: mi-app-secrets
EOF
```

**Promoción por PR** (el CI actualiza el tag de la imagen):

```bash
# CI: update manifest con el nuevo digest, luego PR a main (branch protection)
sed -i "s|image: mi-app:.*|image: mi-app@sha256:${DIGEST}|" apps/mi-app/overlays/prod/deployment.yaml
git commit -m "chore: promote mi-app @ ${SHA}"
git push   # (o gh pr create para que un humano revise)
```

**Rollback GitOps:** `git revert` del commit malo → el operador reconcilia solo. **Seguridad:** commits firmados + permisos mínimos del operador (scoped a namespaces) + secretos cifrados (Sealed/SOPS/External Secrets).

---

## 7. Release automation (SemVer + changelog + GitHub Release)

*Release-please* usa conventional commits para versionar y generar el release:

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    branches: [main]
permissions:
  contents: write
  pull-requests: write
jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          release-type: node
          # bumps MAJOR.MINOR.PATCH desde commits:
          # fix: -> PATCH, feat: -> MINOR, BREAKING CHANGE: -> MAJOR
```

**Convención de commits (ejemplos):**
```text
fix(api): idempotencia en POST /orders      → PATCH
feat(checkout): soporte de cupones          → MINOR
feat!(auth): quitar refresh tokens v1       → MAJOR
```

**Trazabilidad:** el tag del release enlaza el deploy con los commits; el CD (workflow del ejemplo 2) dispara con `push: tags: ["v*.*.*"]`.

---

## 8. Infrastructure pipeline (Terraform + plan en PR + OIDC)

```yaml
# .github/workflows/infra.yml
name: Infra
on:
  pull_request:
    paths: ["infra/**"]
  push:
    branches: [main]
    paths: ["infra/**"]

permissions:
  id-token: write
  contents: read

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/terraform-staging
          aws-region: us-east-1
      - name: Terraform plan
        run: terraform plan -detailed-exitcode   # exit 2 = hay cambios

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'          # solo tras merge
    runs-on: ubuntu-latest
    environment: production                       # gates de prod
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/terraform-prod
          aws-region: us-east-1
      - run: terraform apply -auto-approve
```

**Drift detection** (complemento): un workflow `schedule` que corre `terraform plan -detailed-exitcode` y abre un issue/alerta si hay drift.

---

## 9. Tabla de decisión rápida (problema → patrón/tool)

| Problema | Solución | Dónde |
|---|---|---|
| Compilar distinto en cada entorno | **Build once, deploy many** (artifacts) | Pipeline CI/CD |
| Merge hell / lotes grandes | **Trunk-based development + CI** | Práctica + workflow |
| Release sin downtime y rollback instantáneo | **Blue-Green** | LB/router + pipeline |
| Validar con tráfico real gradual | **Canary** (+ cluster immune system) | Telemetría + pipeline |
| Release por feature sin deploy | **Feature flags / dark launch** | Código + flag service |
| No exponer secretos de prod a cualquier job | **Environments + required reviewers** | GitHub Actions |
| Credenciales estáticas en CI | **OIDC keyless** (roles por entorno) | Actions + IAM trust policy |
| Despliegue con drift y auditoría débil | **GitOps** (ArgoCD/Flux) | Kubernetes |
| Controlar qué dependencias entran | **Dependency Review / SCA** en el PR | Actions |
| Firmar y validar builds | **SLSA + cosign + attest** | Post-build |
| Releases versionados con changelog | **release-please** (conventional commits) | Workflow |
| Despliegue seguro de infra | **Terraform plan en PR + apply tras merge** | Infra pipeline |
| Decidir rollback vs fix forward | **Telemetría/SLO + runbook** | Operación |

---

## 10. Glosario extendido

- **CI / CD / CDel:** Continuous Integration / Continuous Delivery / Continuous Deployment (ver módulo).
- **Deployment vs Release:** instalar una versión vs hacerla visible a usuarios/segmento.
- **Deployment pipeline:** abstracción de build → test → deploy que da feedback seguro y rápido.
- **Trunk-based development:** todos en `main`, lotes pequeños, sin code freezes.
- **Artifact:** paquete inmutable generado una vez y promovido.
- **Environment (GitHub):** destino de deploy con secretos, variables y protection rules.
- **Protection rules:** required reviewers, wait timer, branch/tag policies, custom rules.
- **OIDC:** autenticación keyless entre CI y nube mediante JWT firmado y trust policy.
- **Blue-green / Canary / Rolling:** estrategias de release basadas en entornos/infra.
- **Cluster immune system:** canary donde el monitoreo auto-revierte el rollout.
- **Feature flag / dark launch:** switch en runtime; desplegar invisible y ejercitar con carga real.
- **GitOps:** Git como verdad del estado deseado; operador que pull y reconcilia (declarativo, versionado, pull, reconciliación).
- **ArgoCD / Flux:** operadores GitOps (UI-first vs API-first/CRD-native).
- **Quality gate:** condición automatizada que bloquea el pipeline si no se cumple.
- **SAST / DAST / SCA:** análisis estático, dinámico y de composición de dependencias.
- **DORA metrics:** Deployment Frequency, Lead Time for Changes, Change Failure Rate, MTTR.
- **MTTR / MTBF:** optimizar por tiempo de recuperación, no por prevención absoluta.
- **Rollback / Fix forward:** revertir vs arreglar y desplegar de nuevo.
- **SemVer / Conventional Commits:** versionado semántico y convención de commits que lo automatiza.
- **SLSA:** marco de integridad de la cadena de suministro de software.
