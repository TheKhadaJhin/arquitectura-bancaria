# 04 · Contratos de API: autenticación, saldo y asistente

Presento un borrador de los contratos de autenticación, consulta de saldo y chat con el asistente. Define solicitudes, responsables, respuestas y errores, y complementa el flujo de [autenticación y sesión](03-autenticacion.md).

**Estado:** borrador de contratos; configuración, implementación y pruebas pendientes. Los ejemplos contienen datos ficticios.

**Alcance:** inicio de sesión, canje y renovación de tokens, revocación del refresh token, consulta del saldo contable y mensajes del asistente. Se define la respuesta de preparación de una transferencia; el contrato de ejecución y confirmación sigue pendiente.

## 1. Intercambios y responsables

| Operación | Método y destino | Responsable |
|---|---|---|
| Iniciar login | `GET authorization_endpoint`, abierto en el navegador del sistema. | Proveedor de identidad. |
| Canjear código por tokens | `POST token_endpoint`. | Proveedor de identidad. |
| Renovar tokens | `POST token_endpoint`, con otro `grant_type`. | Proveedor de identidad. |
| Revocar refresh token | `POST revocation_endpoint`. | Proveedor de identidad. |
| Consultar saldo | `GET /v1/accounts/{account_id}/balance`. | Nuestro backend, a través del API Gateway. |
| Enviar un mensaje al asistente | `POST /v1/chat/messages`, propuesto. | Orquestador del backend, con Gemini y funciones autorizadas. |

Los nombres de endpoints OIDC representan URL completas de la configuración confiable del proveedor, verificando su emisor. Las URL reales, el cliente registrado y la redirección quedan pendientes de la selección del proveedor. La revocación deberá estar soportada y configurada. [1][5]

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

La emisión de refresh tokens depende del proveedor. Si exige `offline_access`, se solicitará con el consentimiento correspondiente. La audiencia de la API también debe quedar configurada. [2]

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
| `Authorization` | `Bearer <access_token>` emitido para la API. |
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

La propuesta utiliza un formato común para los errores controlados del gateway y el backend. La app decide por el estado HTTP y `code`; `message` es informativo.

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

## 6. Contrato propuesto del asistente

Este contrato alinea la aplicación con la arquitectura objetivo. No describe todavía un endpoint implementado. El prototipo actual se diferencia en la sección 7.

**Endpoint propuesto:** `POST /v1/chat/messages`, con `Authorization: Bearer <access_token>`, `Content-Type: application/json` y respuestas `Cache-Control: no-store`. El permiso de uso del chat se propone como `chat:use`; las consultas de saldo mantienen `accounts:read` y autorización por cuenta. Los nombres finales de scopes quedan por acordar.

**Solicitud inicial, ejemplo:**

```json
{
  "message": "¿Cuál es el saldo de mi cuenta?",
  "conversation_id": null,
  "account_id": "6b54190c-4e5f-4d34-9c21-2b8ade1f9001"
}
```

`message` es texto no vacío y con longitud limitada. `conversation_id` nulo inicia una conversación cuyo identificador genera el servidor; en mensajes posteriores, el servidor comprueba su propietario. `account_id` es contexto opcional: si la operación requiere cuenta, se selecciona y autoriza antes de consultar datos. El usuario siempre se resuelve desde el token.

La respuesta utiliza un discriminador `type`. El frontend renderiza únicamente los tipos y campos admitidos; el texto del modelo se trata como contenido no confiable.

| `type` | Contenido |
|---|---|
| `text` | `content` con texto informativo o una pregunta para completar datos. No acredita una ejecución financiera. |
| `balance` | `data` con el resultado autorizado de Cuentas: identificador, moneda y saldo contable decimal. |
| `transfer_card` | `data` con una propuesta canónica preparada por el backend y pendiente de confirmación. |

**Respuesta de saldo `200`, ejemplo:**

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

Los valores financieros salen del resultado del módulo bancario. El backend conserva ese resultado estructurado y la app lo muestra directamente; el texto generado no sustituye ni modifica el saldo.

**Tarjeta de preparación `200`, ejemplo:**

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

Los identificadores, la hora y el importe son ficticios. El vencimiento real se calcula en el servidor conforme a una duración pendiente de definir. Si faltan monto o destino, se solicita esa información. El backend resuelve el destinatario contra una fuente autorizada; una etiqueta o un identificador inventado por el modelo no bastan.

`requires_confirmation` es una señal para la interfaz, no una prueba de autorización. Al confirmar, la app enviará `proposal_id` y la clave `X-Idempotency-Key` a un endpoint transaccional separado, cuyo contrato queda pendiente. El servidor verificará pertenencia, vencimiento, contenido, confirmación y step-up exigido, además de riesgo, fondos e idempotencia. Una propuesta solo puede producir una operación efectiva. Cambiar monto o destino exige una nueva propuesta y confirmación.

El catálogo del modelo incluye lectura y preparación; no expone ejecución de transferencias. Repetir un mensaje o una llamada a `prepare_transfer` no debe mover dinero. Si una consulta o ejecución termina con resultado incierto, se consulta su estado por el circuito bancario antes de informar éxito o repetir el efecto.

**Errores adicionales del chat:** se conserva el formato `error.code`, `error.message` y `error.request_id` de la sección 5.

| HTTP | `code` | Comportamiento |
|---|---|---|
| `400` | `INVALID_CHAT_REQUEST` | Mensaje o campos inválidos; corregir la solicitud. |
| `404` | `CONVERSATION_NOT_FOUND` | Conversación inexistente o ajena, con respuesta uniforme. |
| `409` | `CONVERSATION_BUSY` | Ya hay un turno en proceso; evitar mezclar respuestas concurrentes. |
| `429` | `RATE_LIMITED` | Aplicar los límites del chat y respetar `Retry-After` cuando se informe. |
| `502` | `AI_INVALID_RESPONSE` | Respuesta o llamada de función incompatible; no ejecutar la acción. |
| `503` | `AI_UNAVAILABLE` | Servicio, modelo o configuración de IA no disponibles. |
| `504` | `AI_TIMEOUT` | Se superó el tiempo total del turno; informar el fallo sin asegurar un resultado financiero. |

Siguen aplicando `401`/`403` y los errores de cuenta de la sección 5. Los detalles del SDK, nombres internos, trazas y credenciales se registran solo en diagnóstico restringido. El límite total del turno, la cantidad de llamadas a funciones y los reintentos deben ser acotados; un fallo de Gemini no provoca una ejecución financiera automática.

## 7. Relación con el prototipo de UI y Mock API

El prototipo utiliza datos ficticios y rutas simplificadas. La siguiente correspondencia documenta el trabajo necesario para alinearlo; no afirma que esos cambios ya estén implementados.

| Prototipo | Propuesta de integración |
|---|---|
| `POST /api/chat` con `message` y `session_id` | `POST /v1/chat/messages`, con usuario autenticado y `conversation_id` vinculado a ese usuario. |
| `GET /api/v1/accounts/balance`, con `balance` numérico | `GET /v1/accounts/{account_id}/balance`, con ownership y `booked_balance` como texto decimal. |
| `GET /api/v1/accounts/transactions` con movimientos estáticos | Consulta de movimientos con autorización y contrato todavía pendiente. |
| `POST /api/v1/transfers`, con `destination_name` y `amount` | Simulación del prototipo. La ejecución objetivo debe usar una propuesta validada, confirmación e idempotencia; contrato pendiente. |
| `message_type` en la guía y `type` en el orquestador mostrado | Unificar el discriminador en `type` según este borrador, coordinando frontend y backend. |

El adaptador de la API mock transforma las rutas y respuestas sin trasladar las simplificaciones al contrato objetivo. La URL del mock se configura por entorno: si el mock y el orquestador corren en equipos distintos, `localhost` identifica a cada equipo y no al otro servicio.

## 8. Pendientes y verificaciones

| Área | Definición pendiente |
|---|---|
| Proveedor OIDC | URL reales, cliente, redirección, audiencia, scopes, emisión y duración de tokens, rotación y revocación. |
| Acceso a cuentas | Alta del usuario local y política de consulta de cuentas bloqueadas o cerradas. |
| Contrato final | Nombres de campos y códigos, límites de solicitudes, tiempos de espera y reintentos acotados. |
| Origen del saldo | Registro local o core externo; completar apertura y contrapartidas antes de usar datos reales. |
| Chat | Aprobar ruta, scopes, esquema de respuestas, límites, serialización de turnos y conservación del historial. |
| Preparación y ejecución | Almacenamiento de propuestas, resolución del destino, vencimiento, confirmación y contrato transaccional separado. |
| Gemini | Modelo accesible, SDK, timeouts, cuotas y política de datos; verificar respuestas con y sin funciones. |

**Verificaciones previstas:** canje y renovación válidos; rechazo de credenciales inválidas; saldo con formato exacto; cuenta ajena e inexistente con la misma respuesta; indisponibilidad sin saldo inventado; límites de reintento y correlación de errores. Para el asistente se suman conversación ajena, parámetros de funciones inválidos, saldo sin resultado verificable, propuesta vencida, doble confirmación y fallos de Gemini. La ejecución de pruebas y la configuración de servicios están pendientes.

## Relación con el modelo de amenazas

El [análisis STRIDE](05-threat-model.md) traza TM-01–TM-04, TM-09–TM-12 y TM-14–TM-18 hacia estos contratos: identidad, acceso por cuenta, errores, límites y validación de entradas. El límite de solicitudes asociado a `429` requiere configuración y pruebas. La validación de un UUID se complementa con consultas parametrizadas y autorización en el backend.

## Referencias

1. [OpenID Connect Discovery: configuración del proveedor](https://openid.net/specs/openid-connect-discovery-1_0.html).
2. [OpenID Connect Core: autenticación y renovación](https://openid.net/specs/openid-connect-core-1_0.html).
3. [RFC 7636: PKCE](https://www.rfc-editor.org/rfc/rfc7636.html).
4. [RFC 6749: solicitudes, respuestas y errores OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html).
5. [RFC 7009: revocación](https://www.rfc-editor.org/rfc/rfc7009.html).
6. [RFC 9700: seguridad de OAuth 2.0](https://www.rfc-editor.org/rfc/rfc9700.html).
7. [RFC 6750: Bearer tokens y errores de acceso](https://www.rfc-editor.org/rfc/rfc6750.html).
8. [OWASP: autorización de transacciones](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html).
9. [OWASP: privilegios excesivos en asistentes](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/).
