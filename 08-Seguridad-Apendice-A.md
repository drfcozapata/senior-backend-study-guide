# Apéndice A — Módulo 08: Implementaciones de referencia

> Este apéndice complementa [08-Seguridad.md](08-Seguridad.md) con código y configuraciones concretas que puedes adaptar a tus proyectos: flujo PKCE, verificación segura de JWT, políticas IAM de least privilege, manejo de secretos y un glosario extendido.

---

## 1. Flujo Authorization Code + PKCE (S256) con `curl`

Simula el flujo completo contra un Authorization Server (Cognito, Auth0, Okta). La clave del PKCE es que el **código solo viaja junto con el verifier que lo generó**; el servidor comprueba que `SHA256(verifier)` coincide con el `challenge` recibido.

```bash
# 1) Generar code_verifier y code_challenge (PKCE S256)
VERIFIER=$(openssl rand -base64 48 | tr '+/' '-_' | tr -d '=' | cut -c1-128)
CHALLENGE=$(printf '%s' "$VERIFIER" | openssl dgst -sha256 -binary | base64 | tr '+/' '-_' | tr -d '=')

# 2) Redirigir al usuario al endpoint de autorización (obtener el code vía redirect)
open "https://${AUTH_DOMAIN}/authorize?client_id=${CLIENT_ID}&redirect_uri=${REDIRECT}&response_type=code&scope=openid+profile&state=xyz123&code_challenge=${CHALLENGE}&code_challenge_method=S256"

# 3) Intercambiar el code por tokens (code + code_verifier)
curl -X POST "https://${AUTH_DOMAIN}/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&client_id=${CLIENT_ID}&redirect_uri=${REDIRECT}&code=${CODE}&code_verifier=${VERIFIER}" \
  -o tokens.json

cat tokens.json
# → { "access_token": "...", "refresh_token": "...", "id_token": "...", "expires_in": 900 }
```

**Reglas:** el `state` se usa para evitar CSRF en el redirect; el `code_verifier` se guarda en memoria (nunca en `localStorage`) y se borra tras el exchange.

---

## 2. Verificación segura de un JWT (Node.js, `jsonwebtoken`)

La firma es el *AuthZ gate*; aquí además validamos `iss`, `aud`, `exp` y **fijamos el algoritmo** (mitiga `alg:none` y algorithm confusion).

```js
import jwt from "jsonwebtoken";

const JWKS = await fetch("https://${AUTH_DOMAIN}/.well-known/jwks.json").then(r => r.json());
// en producción: `jwks-rsa` / `get-jwks` para cachear y rotar claves por `kid`

const publicKey = jwt.JwksKey(jwks, token).getPublicKey(); // resuelve `kid`

const payload = jwt.verify(token, publicKey, {
  algorithms: ["RS256"],          // FIJA el algoritmo: rechaza alg:none o HS256
  issuer: "https://${AUTH_DOMAIN}", // iss
  audience: `${API_AUDIENCE}`,       // aud = debe ser ESTE servicio
  maxAge: "15m"                      // exp
});

// Autorización fina (AuthZ) en la capa de aplicación:
const order = await getOrder(payload.sub, orderId);
if (order.ownerId !== payload.sub && !hasRole(payload, order.tenant)) {
  throw new ForbiddenError(); // 403 — fail-closed
}
```

**Claves del senior:** nunca confiar en el `alg` del header, validar `aud`, y resolver la clave pública por `kid` con cache de JWKS.

---

## 3. Policies IAM de least privilege (CloudFormation)

### Rol de una Lambda con acceso mínimo a DynamoDB

```yaml
# Servicio: arn:aws:dynamodb:<region>:<acct>:table/orders
OrdersLambdaRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Principal: { Service: lambda.amazonaws.com }
          Action: sts:AssumeRole
    Policies:
      - PolicyName: orders-min
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Effect: Allow
              Action:
                - dynamodb:GetItem
                - dynamodb:Query
              Resource: arn:aws:dynamodb:*:*:table/orders
            - Effect: Allow
              Action: logs:CreateLogGroup
              Resource: "*"
            - Effect: Allow
              Action: logs:CreateLogStream
              Resource: arn:aws:logs:*:*:log-group:/aws/lambda/orders-*
PermissionsBoundary: !Ref OrdersBoundary
```

### Permissions Boundary (que limite el máximo de un rol)

```yaml
OrdersBoundary:
  Type: AWS::IAM::ManagedPolicy
  Properties:
    PolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Action: "*"
          Resource: "*"
          Condition:
            StringLike:
              aws:RequestedRegion: ["us-east-1"]     # solo una región
        - Effect: Deny
          Action: "iam:*"
          Resource: "*"                              # el rol no puede crear otros roles
```

**Resultado:** el rol solo opera en `us-east-1`, nunca escala con IAM, y solo toca `table/orders`. El *permissions boundary* le da el techo aunque después se adjunte una política más amplia.

---

## 4. Manejo de secretos en serverless (IaC)

```yaml
# Secrets Manager con rotación automática (lambda de rotación adjunta)
OrdersDbSecret:
  Type: AWS::SecretsManager::Secret
  Properties:
    Name: /prod/orders/db
    SecretString: '{"username":"orders","password":"rotar-me"}'
    RotationLambdaARN: !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:secret-rotator
    RotationRules: { AutomaticallyAfterDays: 30 }

# Parameter Store SecureString para config simple
UrlParameter:
  Type: AWS::SSM::Parameter
  Properties:
    Name: /prod/orders/internal-url
    Type: SecureString
    Value: "https://internal.orders.cloudns"
```

En la aplicación, se obtiene vía SDK en runtime (no en variables de entorno estáticas ni en el código):

```js
const ssm = new AWS.SSM();
const { Parameter } = await ssm.getParameter({ Name: "/prod/orders/internal-url", WithDecryption: true }).promise();
```

---

## 5. Headers de seguridad en el borde (CloudFront / API Gateway)

```yaml
# CloudFront Distribution ResponseHeadersPolicy
SecurityHeadersPolicy:
  Type: AWS::CloudFront::ResponseHeadersPolicy
  Properties:
    ResponseHeadersPolicyConfig:
      Name: security-headers
      HttpOnly: true
      StrictTransportSecurity:
        Override: true
        AccessControlMaxAgeSec: 31536000
        IncludeSubdomains: true
      ContentTypeOptions:
        Override: true
      FrameOptions: { Override: true, FrameOption: DENY }
      ReferrerPolicy: { Override: true, ReferrerPolicy: no-referrer }
      CustomHeadersEnabled: true
      CustomHeaders:
        - Header: Content-Security-Policy
          Override: true
          Value: "default-src 'self';"   # CSP estricta: nada de inline/remote no declarado
```

---

## 6. Pares de atributos que debes conocer

| Atributo | Significado | Por qué importa |
|---|---|---|
| `Authentication` (AuthN) | ¿Quién es el sujeto? | Base de todo; OAuth 2 solo no basta (usar OIDC) |
| `Authorization` (AuthZ) | ¿Qué puede hacer el sujeto? | **Aquí ocurren los breaches** (broken access control) |
| `Body`/`Object-level` | ¿Tiene permiso *sobre este recurso*? | `orders/123` vs `orders/124` |
| `Authorization: Bearer` | Cómo viaja el token | Nunca en query string |
| `sessionStorage`/`localStorage` | Dónde guardar token | Prefiere memoria + cookies HttpOnly |
| `state` | Anti-CSRF en el flujo OAuth | Evita forgery del callback |
| `code_verifier`/`code_challenge` | PKCE | Previene interceptación del code |
| `kid` | Qué clave firmó | Resuelve la JWKS correcta |
| `jti` | ID único del token | Soporta blocklist/revocación |
| `scope` / `aud` | Alcance y audiencia | Validación imprescindible del token |

---

## 7. Glosario extendido

- **JWT:** *JSON Web Token* — tres segmentos base64url (`header.payload.signature`); firmado, no necesariamente cifrado.
- **JWS / JWE:** si está firmado (JWS) o cifrado (JWE, contenido confidencial).
- **JWKS:** conjunto de claves públicas del emisor, publicado en `/.well-known/jwks.json`.
- **PKCE:** *Proof Key for Code Exchange* — `code_verifier` + `code_challenge`; obligatorio en OAuth 2.1.
- **OIDC:** *OpenID Connect* — capa de autenticación sobre OAuth 2 (ID token).
- **Refresh token rotation:** cada uso emite uno nuevo; el usado queda inválido (reuse → revoke).
- **Permissions boundary:** techo de permisos máximo de un principal en IAM.
- **Deny explícito / implícito:** regla de decisión de AWS IAM.
- **ABAC / RBAC:** autorización por atributos vs por rol.
- **Zero Trust / mTLS:** no confiar dentro de la red; TLS mutuo entre servicios.
- **SBOM:** *Software Bill of Materials* — inventario de dependencias para la cadena de suministro.
- **CIA:** Confidencialidad, Integridad, Disponibilidad.
- **Fail-closed:** negar por defecto ante error.