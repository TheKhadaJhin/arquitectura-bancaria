# Contratos de API: autenticación y saldo

Propongo estos contratos para acordar qué envía la app, quién lo procesa y qué respuesta recibe. Complementan el flujo de [autenticación y sesión](03-autenticacion.md).

**Estado:** borrador para revisar con Diego y el equipo; configuración, implementación y pruebas pendientes. Los ejemplos contienen datos ficticios.

**Alcance:** inicio de sesión, canje y renovación de tokens, revocación del refresh token y consulta del saldo contable de una cuenta. Las transferencias se documentarán aparte.

## 1. Intercambios y responsables

| Operación | Método y destino | Responsable |
|---|---|---|
| Iniciar login | `GET authorization_endpoint`, abierto en el navegador del sistema. | Proveedor de identidad. |
| Canjear código por tokens | `POST token_endpoint`. | Proveedor de identidad. |
| Renovar tokens | `POST token_endpoint`, con otro `grant_type`. | Proveedor de identidad. |
| Revocar refresh token | `POST revocation_endpoint`. | Proveedor de identidad. |
| Consultar saldo | `GET /v1/accounts/{account_id}/balance`. | Nuestro backend, a través del API Gateway. |

Los nombres de endpoints OIDC representan URL completas de la configuración confiable del proveedor, verificando su emisor. Las URL reales, el cliente registrado y la redirección se acordarán al elegirlo. La revocación deberá estar soportada y configurada. [1][5]

Los intercambios de red requieren TLS con verificación del extremo remoto. Los límites y la protección del tramo gateway–backend se concretan en el modelo de amenazas.

## 2. Contrato de inicio de sesión

### Solicitud de autorización

La biblioteca OIDC construye el `GET` al `authorization_endpoint` con estos parámetros: [2][3]

| Parámetro | Valor o criterio |
|---|---|
| `response_type` | `code`. |
| `client_id` | Identificador registrado; `mobile-ios` en los ejemplos. |
| `redirect_uri` | URI registrada y coincidente; por ejemplo, `https://app.example.com/oauth/callback`. |
| `scope` | `openid accounts:read`; `accounts:read` es el permiso de API propuesto. |
| `state` y `nonce` | Valores aleatorios nuevos por intento. |
| `code_challenge` | Desafío derivado del verificador PKCE. |
| `code_challenge_method` | `S256`. |

La emisión de refresh tokens depende del proveedor. Si exige `offline_access`, se solicitará con el consentimiento correspondiente. La audiencia de nuestra API también debe quedar configurada. [2]

**Retorno exitoso:** redirección a la app con `code` y `state`. Un rechazo puede retornar `error=access_denied` y el `state` correspondiente. La biblioteca valida el retorno antes de continuar; los errores de redirección inválida no deben provocar un envío a una URI no confiable.

### Canje del código

La app envía al `token_endpoint` un formulario con `Content-Type: application/x-www-form-urlencoded`: [3][4]

```text
grant_type=authorization_code
client_id=mobile-ios
code={codigo_recibido}
redirect_uri={misma_uri_del_login}
code_verifier={verificador_del_intento}
```

Las líneas muestran campos del formulario: en HTTP se codifican y unen con `&`. Las llaves son marcadores, no valores literales. Para este cliente público no se incluye un secreto embebido.

**Respuesta `200`, ejemplo con refresh token:**

```json
{
  "access_token": "<access-token-ficticio>",
  "token_type": "Bearer",
  "expires_in": 600,
  "refresh_token": "<refresh-token-ficticio>",
  "id_token": "<id-token-ficticio>",
  "scope": "openid accounts:read"
}
```

`expires_in` se expresa en segundos; **600 es ilustrativo**, no una duración acordada. El ID token corresponde al login OIDC; el refresh token solo aparece si fue emitido. La app valida la respuesta según `03-autenticacion.md`. Las respuestas que contienen tokens usan `Cache-Control: no-store` y `Pragma: no-cache`. [2][4]

## 3. Renovación y revocación

Ambas solicitudes van al proveedor como formularios, con la identificación del cliente público que este requiera. [2][5]

| Operación | Campos del formulario | Respuesta esperada |
|---|---|---|
| Renovar | `grant_type=refresh_token`, `refresh_token`; `client_id` según configuración. | `200`: nuevo access token, tipo y expiración; nuevo refresh token bajo la rotación acordada. El ID token es opcional en esta respuesta. |
| Revocar | `token={refresh_token}`, `token_type_hint=refresh_token`; `client_id` según configuración. | `200` si se revocó o el token ya era inválido; el cliente no depende de un cuerpo de respuesta. |

Los errores del proveedor mantienen el formato OAuth. Por ejemplo, un código o refresh token inválido puede responder `400`:

```json
{
  "error": "invalid_grant"
}
```

La app coordina una sola renovación. Una respuesta perdida se trata según la política acordada, sin reutilizar a ciegas el refresh token anterior. El rechazo definitivo requiere un nuevo login. La propuesta de protección utiliza rotación y detección de reutilización. [6]

El logout local elimina las credenciales aunque falle la revocación remota. Invalidar inmediatamente access tokens y cerrar la sesión del navegador son decisiones adicionales, pendientes en el documento de autenticación.

## 4. Contrato de consulta de saldo

**Endpoint propuesto:** `GET /v1/accounts/{account_id}/balance`.

| Entrada | Contrato |
|---|---|
| `account_id` | UUID obligatorio en la ruta. |
| `Authorization` | `Bearer <access_token>` emitido para nuestra API. |
| `Accept` | `application/json`. |
| Cuerpo y parámetros de consulta | Ninguno. |

El backend valida el token, resuelve `(iss, sub)`, exige un usuario local activo y el permiso `accounts:read`, y comprueba que la cuenta pertenezca a ese usuario. El identificador de cuenta no concede acceso por sí mismo.

**Solicitud:**

```http
GET /v1/accounts/6b54190c-4e5f-4d34-9c21-2b8ade1f9001/balance HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token-ficticio>
Accept: application/json
```

**Respuesta:**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
X-Request-ID: b0c48e4f-5c53-47e7-898e-76c3b0554a01
```

```json
{
  "account_id": "6b54190c-4e5f-4d34-9c21-2b8ade1f9001",
  "currency": "ARS",
  "booked_balance": "90000.00"
}
```

| Campo | Significado |
|---|---|
| `account_id` | UUID de la cuenta consultada. |
| `currency` | Moneda de la cuenta; ARS en este alcance. |
| `booked_balance` | Saldo contable como texto decimal, con dos decimales y sin separadores de miles. |

El saldo se calcula desde las entradas `POSTED`, incluida la apertura, tomando una lectura consistente del ledger. La apertura sigue pendiente de diseño en el modelo lógico. Esta respuesta informa el saldo contable; no incorpora reservas ni pretende representar un saldo disponible con reglas aún no modeladas.

La API usa decimales exactos internamente y texto para el importe en JSON. El cliente debe conservar esa precisión. Esta consulta es de lectura: no requiere una clave de idempotencia ni crea movimientos. No ofrece consultas históricas en esta versión.

## 5. Errores y comportamiento del cliente

Para los errores controlados de nuestra API propongo este cuerpo, compartido por gateway y backend. La app decide por el estado HTTP y `code`; `message` es informativo.

**Ejemplo de `404`:**

```json
{
  "error": {
    "code": "ACCOUNT_NOT_FOUND",
    "message": "No se encontró la cuenta solicitada.",
    "request_id": "b0c48e4f-5c53-47e7-898e-76c3b0554a01"
  }
}
```

| HTTP | `code` | Criterio y acción |
|---|---|---|
| `400` | `INVALID_ACCOUNT_ID` | UUID inválido. Corregir la solicitud. |
| `401` | `UNAUTHORIZED` | Token ausente, inválido o vencido. Evaluar renovación; como máximo un reintento por ese motivo. |
| `403` | `FORBIDDEN` | Falta permiso funcional o el usuario local no está habilitado. No renovar por este error. |
| `404` | `ACCOUNT_NOT_FOUND` | Cuenta inexistente o ajena: mismo estado y mensaje. |
| `429` | `RATE_LIMITED` | Respetar `Retry-After` cuando se informe. |
| `503` | `SERVICE_UNAVAILABLE` | No se puede obtener un saldo confiable. Mostrar indisponibilidad; nunca reemplazarla por saldo cero. |
| `500` | `INTERNAL_ERROR` | Fallo inesperado. Mostrar un mensaje genérico y conservar la referencia de soporte. |

Los `401` incluyen `WWW-Authenticate: Bearer`; para un token presentado pero inválido, se añade `error="invalid_token"`. La falta del scope requerido usa `403` con `error="insufficient_scope"` en ese encabezado. [7]

Respuestas de saldo y errores: `Cache-Control: no-store`. La infraestructura confiable genera `X-Request-ID`, lo propaga y lo repite en `error.request_id`. Los errores no exponen tokens, consultas SQL ni datos de otras cuentas. Una falla de transporte puede no tener cuerpo JSON y no demuestra que la sesión sea inválida.

## 6. Pendientes y verificaciones

| Pendiente | Qué debemos acordar |
|---|---|
| Proveedor OIDC | URL reales, cliente, redirección, audiencia, scopes, emisión y duración de tokens, rotación y revocación. |
| Acceso a cuentas | Alta del usuario local y política de consulta de cuentas bloqueadas o cerradas. |
| Contrato final | Nombres de campos y códigos, límites de solicitudes, tiempos de espera y reintentos acotados. |
| Origen del saldo | Registro local o core externo; completar apertura y contrapartidas antes de usar datos reales. |

**Verificaciones previstas:** canje y renovación válidos; rechazo de credenciales inválidas; saldo con formato exacto; cuenta ajena e inexistente con la misma respuesta; indisponibilidad sin saldo inventado; límites de reintento y correlación de errores. No se ejecutaron pruebas ni se configuraron servicios.

## Relación con el modelo de amenazas

El [análisis STRIDE](05-threat-model.md) traza TM-01–TM-04 y TM-09–TM-12 hacia estos contratos: identidad, acceso por cuenta, errores, límites y validación de entradas. Un `429` documentado no implementa el límite: faltan su configuración y pruebas. La validación de un UUID se complementa con consultas parametrizadas y autorización en el backend.

## Referencias

1. [OpenID Connect Discovery: configuración del proveedor](https://openid.net/specs/openid-connect-discovery-1_0.html).
2. [OpenID Connect Core: autenticación y renovación](https://openid.net/specs/openid-connect-core-1_0.html).
3. [RFC 7636: PKCE](https://www.rfc-editor.org/rfc/rfc7636.html).
4. [RFC 6749: solicitudes, respuestas y errores OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html).
5. [RFC 7009: revocación](https://www.rfc-editor.org/rfc/rfc7009.html).
6. [RFC 9700: seguridad de OAuth 2.0](https://www.rfc-editor.org/rfc/rfc9700.html).
7. [RFC 6750: Bearer tokens y errores de acceso](https://www.rfc-editor.org/rfc/rfc6750.html).
