# Propuesta de arquitectura bancaria

Preparé esta base para revisar con Diego y el equipo la arquitectura, el registro de movimientos, la autenticación y los contratos iniciales de la aplicación. La idea es dejar explícitos los supuestos y ajustar las decisiones antes de implementar.

**Estado:** cinco documentos preparados para revisión. La implementación y las pruebas están pendientes; los controles de seguridad descritos todavía deben implementarse y verificarse.

## Documentación

| Documento | Contenido |
|---|---|
| [Arquitectura general](01-arquitectura.md) | App móvil, proveedor de identidad, API Gateway, backend modular, PostgreSQL y límites de confianza. |
| [Modelo relacional](02-modelo-relacional.md) | Entidades, claves, ledger de partida doble, atomicidad, idempotencia, concurrencia y reversas. |
| [Autenticación](03-autenticacion.md) | OAuth 2.0 / OIDC con PKCE, access y refresh tokens, cierre de sesión y almacenamiento en Keychain de iOS. |
| [Contratos de API](04-contratos-api.md) | Operaciones de autenticación con el proveedor de identidad y contrato de consulta de saldo, con respuestas y errores. |
| [Modelo de amenazas](05-threat-model.md) | Análisis STRIDE, límites de confianza, amenazas, controles propuestos y verificaciones previstas. |

El orden de la tabla sirve como recorrido de lectura. El análisis de amenazas acompaña las decisiones de todos los documentos y se actualiza cuando cambia el diseño.

## Alcance inicial

La propuesta cubre autenticación móvil, consulta de saldo contabilizado y el diseño de transferencias internas entre dos cuentas distintas y activas, en ARS, sin comisiones, retenciones ni descubierto. El contrato de la API de transferencias queda pendiente. Los ejemplos utilizan datos ficticios.

Por ahora, PostgreSQL se considera la fuente de verdad de los movimientos. Tenemos que confirmar si corresponde al proyecto o si las operaciones se ejecutan en un core bancario externo: esa definición puede cambiar la confirmación y la conciliación de las transferencias.

## Decisiones propuestas

- Mantener un backend modular detrás del API Gateway. El backend valida identidad, permisos sobre cada cuenta y reglas de negocio.
- Delegar la autenticación en un proveedor de identidad mediante Authorization Code con PKCE; mantener el access token en memoria y el refresh token en Keychain.
- Registrar las operaciones en `transactions` y sus asientos en `ledger_entries`, con partida doble y saldos reconstruibles desde el historial confirmado. Las correcciones generan operaciones compensatorias sin modificar los asientos originales.
- Confirmar cada operación y sus asientos en una transacción SQL. Usar idempotencia por usuario, comparación de la solicitud y un protocolo común de bloqueo y comprobación de fondos.
- Mantener las amenazas STRIDE vinculadas con controles y evidencias de verificación. Las amenazas identificadas siguen abiertas.

Las restricciones, los casos de error y las verificaciones previstas se detallan en cada documento.

## Próximos pasos

1. Acordar con Diego el stack del backend y de la app móvil.
2. Preparar la estructura de los repositorios, los linters y las protecciones de ramas.
3. Resolver con el equipo los supuestos pendientes: core bancario, proveedor de identidad, políticas de sesión, asientos de apertura y cuentas internas del ledger.
4. Asignar responsables a las amenazas y llevar sus controles y pruebas a la implementación de cada flujo.
