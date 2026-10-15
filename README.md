# Propuesta de arquitectura bancaria

En este repositorio presento la propuesta de arquitectura para la aplicación bancaria, junto con el modelo de datos, los flujos de autenticación, los contratos de API, el asistente con Gemini y el análisis de amenazas. Cada documento detalla las decisiones de diseño, su alcance y los puntos pendientes antes de implementar.

**Estado:** documentación de diseño actualizada al 14 de octubre de 2026. Se incorporan el asistente con Gemini y HashiCorp Vault. El prototipo del asistente se trabaja por separado con datos ficticios; este repositorio no contiene su implementación. Los controles descritos siguen pendientes de implementación y verificación, al igual que el scaffolding, CI y las protecciones de ramas. El esquema detallado de eventos/outbox y el contrato de ejecución de transferencias siguen abiertos.

## Documentación

| Documento | Contenido |
|---|---|
| [Arquitectura general](01-arquitectura.md) | App móvil, identidad, API Gateway/WAF, backend modular, motor de riesgo, asistente con Gemini, persistencia e integración con sistemas externos. |
| [Modelo relacional](02-modelo-relacional.md) | Entidades, claves, ledger de partida doble, atomicidad, idempotencia, concurrencia, reversiones y separación de las propuestas del asistente. |
| [Autenticación](03-autenticacion.md) | OAuth 2.0 / OIDC con PKCE, manejo de tokens, almacenamiento seguro, cierre de sesión y acceso del asistente. |
| [Contratos de API](04-contratos-api.md) | Autenticación, saldo y borrador del chat, con respuestas estructuradas y diferencias del prototipo. |
| [Modelo de amenazas](05-threat-model.md) | Análisis STRIDE, límites de confianza y verificaciones, incluidos el asistente y los secretos. |
| [Stack y scaffolding](06-stack-y-scaffolding.md) | Tecnologías propuestas, Gemini, HashiCorp Vault, Kafka y estructura inicial de los proyectos. |

## Alcance inicial

La aplicación está prevista para **Android e iOS**. El alcance inicial incluye autenticación móvil, consulta de saldo contable y diseño de transferencias internas entre cuentas distintas y activas, en ARS, sin comisiones, retenciones ni descubierto. El asistente permite consultar saldo y preparar una propuesta de transferencia para confirmación. El contrato de ejecución de transferencias queda pendiente. Los ejemplos utilizan datos ficticios.

El flujo interno supone que PostgreSQL registra ambos lados de la transferencia. Queda pendiente definir su relación con el core bancario y qué sistema será la fuente oficial de los movimientos. Esa decisión determina cómo se confirman y concilian las operaciones externas.

## Decisiones propuestas

- **Backend modular:** concentrar las reglas de negocio en una aplicación detrás del API Gateway. Validar identidad, permisos sobre cada cuenta y condiciones de la operación.
- **Autenticación:** utilizar un proveedor de identidad con Authorization Code + PKCE. Mantener el access token en memoria y proteger el refresh token con Keychain en iOS y una solución apoyada en Keystore para Android.
- **Registro contable:** utilizar un ledger de partida doble, con saldos reconstruibles y asientos confirmados que no se modifican. Las correcciones se registran mediante operaciones compensatorias.
- **Transferencias internas:** confirmar la operación y sus asientos en una misma transacción SQL, con idempotencia, control de concurrencia y comprobación de fondos.
- **Historial y eventos:** incorporar eventos append-only y una outbox en la transacción local. Publicar en Kafka después del commit y procesar los eventos con consumidores idempotentes. Saga queda sujeta a la necesidad de coordinar pasos distribuidos.
- **Protección y riesgo:** evaluar el riesgo antes de autorizar movimientos e incorporar los controles de dispositivo y autenticación adicional definidos en la arquitectura. Separar el acceso a datos personales sensibles mediante un Data Vault cifrado y permisos específicos para las claves.
- **Asistente:** incorporar un orquestador dentro del backend y Gemini como servicio de IA. Las consultas pasan por los módulos autorizados. La preparación de una transferencia devuelve una tarjeta; la ejecución requiere confirmación verificada por el servidor y conserva todos los controles financieros.
- **Secretos y claves:** utilizar HashiCorp Vault en la propuesta para las credenciales técnicas y definir la configuración criptográfica necesaria. El Data Vault de datos personales mantiene almacenamiento y permisos separados.
- **Seguridad:** mantener el análisis STRIDE vinculado a los flujos, controles y pruebas. Los controles descritos requieren implementación y verificación.
- **Stack:** evaluar Python + FastAPI para backend y React Native + TypeScript para mobile. La selección final queda pendiente.
