# Propuesta de arquitectura bancaria

En este repositorio presento la propuesta de arquitectura para la aplicación bancaria, junto con el modelo de datos, los flujos de autenticación, los contratos de API y el análisis de amenazas. Cada documento detalla las decisiones de diseño, su alcance y los puntos pendientes antes de implementar.

**Estado:** documentación de diseño. La implementación, el scaffolding, la configuración de CI y las protecciones de ramas están pendientes. Las ampliaciones de la arquitectura general todavía deben incorporarse al detalle de las secciones 02–06.

## Documentación

| Documento | Contenido |
|---|---|
| [Arquitectura general](01-arquitectura.md) | App móvil, identidad, API Gateway/WAF, backend modular, motor de riesgo, persistencia e integración con sistemas externos. |
| [Modelo relacional](02-modelo-relacional.md) | Entidades, claves, ledger de partida doble, atomicidad, idempotencia, concurrencia y reversiones. |
| [Autenticación](03-autenticacion.md) | OAuth 2.0 / OIDC con PKCE, manejo de tokens, almacenamiento seguro y cierre de sesión. |
| [Contratos de API](04-contratos-api.md) | Intercambios de autenticación y consulta de saldo, con solicitudes, respuestas y errores. |
| [Modelo de amenazas](05-threat-model.md) | Análisis STRIDE, límites de confianza, controles propuestos y verificaciones previstas. |
| [Stack y scaffolding](06-stack-y-scaffolding.md) | Tecnologías candidatas, estructura de los proyectos y configuración inicial de desarrollo. |

## Alcance inicial

La aplicación está prevista para **Android e iOS**. El alcance inicial incluye autenticación móvil, consulta de saldo contable y diseño de transferencias internas entre cuentas distintas y activas, en ARS, sin comisiones, retenciones ni descubierto. El contrato de transferencias queda pendiente. Los ejemplos utilizan datos ficticios.

El flujo interno supone que PostgreSQL registra ambos lados de la transferencia. Queda pendiente definir su relación con el core bancario y qué sistema será la fuente oficial de los movimientos. Esa decisión determina cómo se confirman y concilian las operaciones externas.

## Decisiones propuestas

- **Backend modular:** concentrar las reglas de negocio en una aplicación detrás del API Gateway. Validar identidad, permisos sobre cada cuenta y condiciones de la operación.
- **Autenticación:** utilizar un proveedor de identidad con Authorization Code + PKCE. Mantener el access token en memoria y proteger el refresh token con Keychain en iOS y una solución apoyada en Keystore para Android.
- **Registro contable:** utilizar un ledger de partida doble, con saldos reconstruibles y asientos confirmados que no se modifican. Las correcciones se registran mediante operaciones compensatorias.
- **Transferencias internas:** confirmar la operación y sus asientos en una misma transacción SQL, con idempotencia, control de concurrencia y comprobación de fondos.
- **Historial y eventos:** incorporar eventos append-only y una outbox en la transacción local. Publicar en Kafka después del commit y procesar los eventos con consumidores idempotentes. Saga queda sujeta a la necesidad de coordinar pasos distribuidos.
- **Protección y riesgo:** evaluar el riesgo antes de autorizar movimientos e incorporar los controles de dispositivo y autenticación adicional definidos en la arquitectura. Separar el acceso a datos personales sensibles mediante un Data Vault cifrado y permisos específicos para las claves.
- **Seguridad:** mantener el análisis STRIDE vinculado a los flujos, controles y pruebas. Los controles descritos requieren implementación y verificación.
- **Stack:** evaluar Python + FastAPI para backend y React Native + TypeScript para mobile. La selección final queda pendiente.

