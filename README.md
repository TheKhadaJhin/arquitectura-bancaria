# Arquitectura de la aplicación bancaria

Este repositorio reúne el diseño de la aplicación y sirve como material de apoyo para el equipo. Explica cómo se conectan los componentes, cómo se registra el dinero y qué controles debe respetar cada flujo.

**Estado al 16 de octubre de 2026:** documentación de diseño. Los controles, contratos y scaffolding requieren implementación y verificación. La demo del asistente se desarrolla por separado con datos ficticios; sus simulaciones no acreditan esos controles en producción.

## Guía de lectura

Para una primera lectura, seguir el orden de los documentos. Después, cada sección puede consultarse por separado según la tarea.

| Documento | Qué explica |
|---|---|
| [01 · Arquitectura](01-arquitectura.md) | Componentes, responsabilidades y recorrido de una transferencia y del asistente. |
| [02 · Modelo relacional](02-modelo-relacional.md) | Entidades y claves; partida doble, atomicidad, idempotencia y concurrencia. |
| [03 · Autenticación](03-autenticacion.md) | Login con OAuth/OIDC y PKCE, tokens, almacenamiento seguro y sesión. |
| [04 · Contratos de API](04-contratos-api.md) | Solicitudes, respuestas y errores de autenticación, saldo y chat. |
| [05 · STRIDE](05-threat-model.md) | Qué puede fallar o ser atacado, cómo se propone protegerlo y cómo comprobarlo. |
| [06 · Stack y scaffolding](06-stack-y-scaffolding.md) | Tecnologías propuestas, estructura de los proyectos y preparación inicial. |

## Alcance y decisiones

La app está prevista para **Android e iOS**. El alcance inicial contempla login, saldo contable y transferencias internas en ARS entre cuentas distintas y activas, sin comisiones, retenciones ni descubierto. El asistente consulta saldo y prepara propuestas que el usuario debe confirmar.

La base del diseño combina backend modular, PostgreSQL con ledger de partida doble, outbox y Kafka para eventos, Gemini para el asistente y HashiCorp Vault para secretos y operaciones con claves. La identidad se delega en un proveedor OIDC. Cada sección explica el motivo y los límites de estas decisiones.

**Pendientes principales:** definir la relación del ledger con el core bancario, completar cuentas de apertura y tablas de eventos/outbox, cerrar el contrato de ejecución de transferencias y acordar stack, proveedores y configuración operativa. Los ejemplos internos suponen que PostgreSQL registra ambos lados de la transferencia.
