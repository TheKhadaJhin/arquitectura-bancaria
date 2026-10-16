# 04 · Contratos de API

Un contrato define qué envía el cliente, qué valida el servidor y cómo responde. Este borrador cubre autenticación, saldo y asistente; permite alinear frontend y backend antes de implementar.

**Estado:** contratos propuestos, con ejemplos ficticios. El endpoint de ejecución y confirmación de transferencias sigue pendiente. La demo se desarrolla por separado y sus rutas no constituyen el contrato objetivo.

## 1. Operaciones y responsables

| Operación | Método y destino | Responsable |
|---|---|---|
| Login | `GET authorization_endpoint`, mediante navegador del sistema. | Proveedor de identidad. |
| Canje y renovación | `POST token_endpoint`. | Proveedor de identidad. |
| Revocación | `POST revocation_endpoint`. | Proveedor de identidad. |
| Saldo | `GET /v1/accounts/{account_id}/balance`. | Backend, a través del gateway. |
| Chat | `POST /v1/chat/messages`. | Orquestador del backend. |

Los endpoints OIDC son URL completas obtenidas de configuración confiable, verificando el emisor. Cliente, redirecciones y URL reales dependen del proveedor elegido. Todos los intercambios requieren TLS con verificación del extremo remoto.

## 2. Intercambios con el proveedor de identidad

### Login y canje del código

La biblioteca OIDC construye el `GET authorization_endpoint`:

| Parámetro | Valor o criterio |
|---|---|
| `response_type` | `code`. |
| `client_id` y `redirect_uri` | Cliente y retorno registrados; ejemplos: `mobile-ios` y `https://app.example.com/oauth/callback`. |
| `scope` | `openid accounts:read`; el permiso de API es propuesto. Solicitar `offline_access` si el proveedor lo exige para emitir refresh token. |
| `state` y `nonce` | Valores nuevos por intento. |
| `code_challenge` y `code_challenge_method` | Desafío PKCE y `S256`. |

La audiencia de la API debe configurarse. El retorno exitoso contiene `code` y `state`; un rechazo puede retornar `error=access_denied`. La biblioteca valida el retorno antes de continuar, como describe [autenticación](03-autenticacion.md#2-login-con-authorization-code--pkce). Una URI inválida no debe recibir la redirección.

El canje usa `POST token_endpoint` con `Content-Type: application/x-www-form-urlencoded`:

```text
grant_type=authorization_code
client_id=mobile-ios
code={codigo_recibido}
redirect_uri={misma_uri_del_login}
code_verifier={verificador_del_intento}
```

Son campos de formulario: en HTTP se codifican y unen con `&`. Las llaves son marcadores. El cliente público no envía un secreto embebido.

**Respuesta `200`, ejemplo:**

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

`expires_in` está en segundos; 600 es ilustrativo. El refresh token solo aparece si fue emitido. Las respuestas con tokens llevan `Cache-Control: no-store` y `Pragma: no-cache`.

### Renovación y revocación

También usan formularios y la identificación del cliente público que requiera el proveedor.

| Operación | Campos | Resultado |
|---|---|---|
| Renovar | `grant_type=refresh_token`, `refresh_token`; `client_id` según configuración. | `200`: nuevo access token, tipo y expiración; nuevo refresh token bajo la rotación acordada. El ID token es opcional. |
| Revocar | `token={refresh_token}`, `token_type_hint=refresh_token`; `client_id` según configuración. | `200` si se revocó o ya era inválido; no depender de un cuerpo de respuesta. |

Los errores conservan el formato OAuth: por ejemplo, `400` con `{"error":"invalid_grant"}` para un código o refresh token inválido. La coordinación de renovaciones, recuperación de respuestas perdidas y logout se explica en [03](03-autenticacion.md#4-renovación-y-almacenamiento).

## 3. Consulta de saldo

**`GET /v1/accounts/{account_id}/balance`** requiere UUID en la ruta, access token y `Accept: application/json`. No recibe cuerpo ni parámetros de consulta.

El backend resuelve al usuario desde `(iss, sub)`, exige estado activo, permiso `accounts:read` y acceso a la cuenta. Conocer su UUID no concede autorización.

```http
GET /v1/accounts/6b54190c-4e5f-4d34-9c21-2b8ade1f9001/balance HTTP/1.1
Host: api.example.com
Authorization: Bearer <access-token-ficticio>
Accept: application/json
```

**Respuesta `200`:**

```json
{
  "account_id": "6b54190c-4e5f-4d34-9c21-2b8ade1f9001",
  "currency": "ARS",
  "booked_balance": "90000.00"
}
```

`booked_balance` es texto decimal con dos decimales y sin separadores de miles. Se calcula desde entradas `POSTED`, incluida la apertura, con una lectura consistente. La apertura y sus contrapartidas aún deben completarse en el [modelo](02-modelo-relacional.md#2-partida-doble-y-saldo).

Este **saldo contable** no incluye reservas ni representa un saldo disponible con reglas todavía no modeladas. La API y el cliente conservan precisión decimal. La consulta no requiere clave de idempotencia ni crea movimientos; las consultas históricas quedan fuera de esta versión.

## 4. Errores comunes

Saldo, chat y errores del backend usan JSON, `Cache-Control: no-store` y `X-Request-ID`, generado por infraestructura confiable. El identificador se repite en `error.request_id`. La app decide por estado HTTP y `code`; `message` es informativo.

```json
{
  "error": {
    "code": "ACCOUNT_NOT_FOUND",
    "message": "No se encontró la cuenta solicitada.",
    "request_id": "b0c48e4f-5c53-47e7-898e-76c3b0554a01"
  }
}
```

| HTTP | `code` | Acción del cliente |
|---|---|---|
| `400` | `INVALID_ACCOUNT_ID` | Corregir UUID inválido. |
| `401` | `UNAUTHORIZED` | Evaluar renovación; máximo un reintento por ese motivo. |
| `403` | `FORBIDDEN` | Falta permiso o usuario no habilitado; no renovar. |
| `404` | `ACCOUNT_NOT_FOUND` | Cuenta inexistente o ajena: mismo estado y mensaje. |
| `429` | `RATE_LIMITED` | Respetar `Retry-After` cuando se informe. |
| `503` | `SERVICE_UNAVAILABLE` | Mostrar indisponibilidad; nunca reemplazarla por saldo cero. |
| `500` | `INTERNAL_ERROR` | Mostrar error genérico y referencia para soporte. |

Los `401` incluyen `WWW-Authenticate: Bearer`, con `error="invalid_token"` si se presentó un token inválido. La falta de scope usa `403` y `error="insufficient_scope"` en ese encabezado.

Una falla de transporte puede no traer JSON y no demuestra que la sesión sea inválida. Los detalles técnicos quedan en diagnóstico restringido; las respuestas no exponen SQL, trazas, tokens ni datos de otras cuentas.

## 5. Asistente

**`POST /v1/chat/messages`** usa access token y `Content-Type: application/json`. Se propone `chat:use`; las funciones de saldo mantienen `accounts:read` y autorización por cuenta. Los nombres finales de scopes siguen pendientes.

**Solicitud inicial:**

```json
{
  "message": "¿Cuál es el saldo de mi cuenta?",
  "conversation_id": null,
  "account_id": "6b54190c-4e5f-4d34-9c21-2b8ade1f9001"
}
```

`message` debe ser texto no vacío y limitado. `conversation_id: null` inicia una conversación identificada por el servidor; en accesos posteriores se comprueba su propietario. `account_id` es contexto opcional y siempre se autoriza antes de usarlo. El usuario se obtiene del token.

| `type` de respuesta | Contenido y uso |
|---|---|
| `text` | `content`: información o pregunta para completar datos; no acredita una ejecución. |
| `balance` | `data`: resultado autorizado de Cuentas. |
| `transfer_card` | `data`: propuesta preparada por el backend, pendiente de confirmación. |

La app renderiza solo tipos y campos admitidos. Trata el texto del modelo como contenido no confiable y muestra los valores financieros estructurados del backend.

**Saldo `200`:**

```json
{
  "conversation_id": "1e5e5e41-299f-49d6-9168-85bf1f7e573d",
  "type": "balance",
  "data": {
    "account_id": "6b54190c-4e5f-4d34-9c21-2b8ade1f9001",
    "currency": "ARS",
    "booked_balance": "90000.00"
  }
}
```

**Propuesta de transferencia `200`:**

```json
{
  "conversation_id": "1e5e5e41-299f-49d6-9168-85bf1f7e573d",
  "type": "transfer_card",
  "data": {
    "proposal_id": "cf079dd9-519b-412a-9c3d-96e6ea240732",
    "source_account_id": "6b54190c-4e5f-4d34-9c21-2b8ade1f9001",
    "destination_account_id": "39abbe07-b91f-4aaf-8ec2-7b0cf3670d94",
    "destination_label": "Destinatario de prueba",
    "amount": "50000.00",
    "currency": "ARS",
    "expires_at": "2026-10-14T18:30:00Z",
    "requires_confirmation": true
  }
}
```

Fecha e importe son ficticios; el servidor calcula el vencimiento real. Si falta monto o destino, se pregunta. El destinatario se resuelve contra una fuente autorizada; no basta una etiqueta inventada por Gemini.

`requires_confirmation` orienta a la interfaz, pero no prueba autorización. El usuario revisará la tarjeta y confirmará enviando `proposal_id` y `X-Idempotency-Key` al endpoint transaccional **aún por definir**. El servidor verifica propuesta, propietario, vigencia, datos, confirmación, riesgo, step-up, fondos e idempotencia. Una propuesta solo puede producir una operación efectiva; cambiar monto o destino exige otra confirmación sobre una nueva propuesta.

Gemini solo puede consultar o preparar. Repetir el mensaje o `prepare_transfer` no mueve dinero. Ante un resultado financiero incierto, se consulta su estado por el circuito bancario antes de informar éxito o repetir el efecto.

### Errores del chat

Conservan el formato anterior y también aplican `401`, `403`, `429` y los errores de cuenta.

| HTTP | `code` | Significado |
|---|---|---|
| `400` | `INVALID_CHAT_REQUEST` | Mensaje o campos inválidos. |
| `404` | `CONVERSATION_NOT_FOUND` | Conversación inexistente o ajena, con respuesta uniforme. |
| `409` | `CONVERSATION_BUSY` | Hay un turno en proceso; no mezclar respuestas. |
| `502` | `AI_INVALID_RESPONSE` | Respuesta o función incompatible; no ejecutar la acción. |
| `503` | `AI_UNAVAILABLE` | Servicio, modelo o configuración no disponibles. |
| `504` | `AI_TIMEOUT` | Venció el tiempo total del turno. |

Tiempo total, funciones por turno y reintentos deben ser acotados. Un fallo de Gemini no autoriza una ejecución automática ni permite inventar resultados.

## 6. Pendientes y validación

Faltan configuración OIDC, política de acceso a cuentas bloqueadas/cerradas, nombres finales de campos y scopes, límites, historial del chat y contrato de confirmación. También deben cerrarse origen del saldo, apertura, modelo de Gemini y política de datos.

Las pruebas deben cubrir los ejemplos y sus errores, precisión decimal, aislamiento entre usuarios, confirmaciones duplicadas y fallos del proveedor. [STRIDE](05-threat-model.md) reúne los escenarios. Estos contratos no afirman que la demo ya implemente esas garantías.

## Para profundizar

- OpenID Connect: [Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) y [Core](https://openid.net/specs/openid-connect-core-1_0.html).
- IETF: [solicitudes y errores OAuth](https://www.rfc-editor.org/rfc/rfc6749.html), [revocación](https://www.rfc-editor.org/rfc/rfc7009.html) y [Bearer tokens](https://www.rfc-editor.org/rfc/rfc6750.html).
- OWASP: [autorización de transacciones](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html).
