# Propuesta de arquitectura bancaria

Preparé esta base para revisar con Diego y el equipo la arquitectura de la aplicación, el modelo de datos y las reglas de las transferencias. La idea es dejar explícitos los supuestos y ajustar las decisiones antes de implementar.

**Estado:** propuesta inicial. Implementación y pruebas pendientes.

## Documentación

| Documento | Contenido |
|---|---|
| [Arquitectura general](01-arquitectura.md) | App móvil, proveedor de identidad, API Gateway, backend modular, PostgreSQL y límites de confianza. |
| [Modelo relacional](02-modelo-relacional.md) | Entidades, campos, claves, atomicidad, idempotencia, concurrencia y verificaciones previstas. |
| [Autenticación](03-autenticacion.md) |  |
| [Contrato](04-contratos-api.md) |  |
| [Threat Model con STRIDE](05-threat-model.md) |  |

## Alcance inicial

El modelo considera transferencias internas entre dos cuentas distintas y activas, en ARS, sin comisiones, retenciones ni descubierto. Los ejemplos utilizan datos ficticios.

Por ahora, la propuesta supone que nuestra PostgreSQL registra los movimientos de dinero. Tenemos que confirmar si este supuesto corresponde al proyecto o si las operaciones se ejecutan en un core bancario externo. Esa definición puede cambiar los estados, la confirmación y la conciliación de las transferencias.

## Decisiones propuestas

- Separar el proveedor de identidad de la aplicación y validar en el backend la identidad, los permisos y las reglas de negocio.
- Organizar el backend como una aplicación modular, con responsabilidades separadas para usuarios, cuentas y movimientos.
- Representar cada operación en `transactions` y sus efectos sobre las cuentas en `ledger_entries`.
- Confirmar la operación y sus movimientos dentro de una misma transacción SQL.
- Resolver los reintentos con una clave de idempotencia por usuario y comparar el contenido de la solicitud.
- Controlar las operaciones simultáneas mediante un protocolo común de bloqueo y comprobación de fondos.

Los detalles, las restricciones y las pruebas necesarias están documentados en el modelo relacional.

## Próximos pasos

1. Documentar el flujo de autenticación, la renovación de sesión y el almacenamiento seguro en el dispositivo.
2. Definir los contratos iniciales de autenticación y consulta de saldo.
3. Revisar el stack y preparar la estructura de los proyectos.
4. Revisar con el equipo los supuestos, las amenazas y las decisiones pendientes antes de implementar.

