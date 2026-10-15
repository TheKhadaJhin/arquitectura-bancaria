# 06 · Stack y estructura inicial de los proyectos

**Alcance confirmado:** app para Android e iOS. PostgreSQL está indicado en el requerimiento.

**Estado:** propuesta de stack y estructura inicial. Se incorporan Gemini, HashiCorp Vault y Kafka a las decisiones documentadas. Queda pendiente acordar el framework móvil, el backend, la organización de los repositorios y el almacenamiento seguro de Android. La configuración y verificación de proyectos, CI y protecciones de ramas están pendientes.

## Criterio de elección

Propongo seleccionar el stack según los estándares del equipo, la cobertura de Android e iOS, la compatibilidad con el proveedor de identidad y el mantenimiento del proyecto. La tecnología elegida debe permitir aplicar las reglas del ledger y los controles ya documentados.

## Stack propuesto

| Componente | Propuesta | Motivo y estado |
|---|---|---|
| Comunicación app–backend | API REST sobre HTTPS, con representaciones JSON | Propuesta coherente con los contratos preparados. La misma API atendería clientes Android e iOS. |
| Base de datos | PostgreSQL | Incluido en los requerimientos. El papel de PostgreSQL frente a un posible core bancario externo sigue pendiente de confirmación. |
| Backend | Python + FastAPI | Utilizado en el prototipo separado del orquestador. Candidato para el backend modular objetivo, sujeto al estándar del equipo. |
| Persistencia | SQLAlchemy + Alembic | Candidatos para manejar acceso a datos, transacciones explícitas y migraciones versionadas si se elige Python. |
| App Android e iOS | React Native + TypeScript | Candidato para compartir parte del código. Validar experiencia del equipo, SDK de identidad, funciones nativas y mantenimiento en ambas plataformas. |
| Almacenamiento de sesión | Keychain en iOS; solución de cifrado apoyada en Android Keystore por evaluar en Android | El criterio de protección es común; biblioteca, configuración y pruebas son específicas de cada plataforma. |
| Identidad y API Gateway | Selección pendiente | Mantener OAuth 2.0 / OIDC y las responsabilidades descritas en la arquitectura. Confirmar primero las soluciones existentes en la empresa. |
| Servicio de IA | Gemini | Servicio elegido para el asistente. La versión concreta del modelo se configura y valida en el entorno. |
| SDK de IA para Python | Google GenAI SDK (`google-genai`) | Biblioteca oficial recomendada. Adaptar el cliente del prototipo y verificar las funciones antes de cerrar la integración. [13] |
| Secretos y claves | HashiCorp Vault | Herramienta incluida en la propuesta. Concretar motores, identidades, políticas, rotación y recuperación; evaluar Transit para la función criptográfica. [14][15] |
| Eventos | Apache Kafka + Outbox Pattern | Kafka distribuye eventos; la outbox registra la intención de publicación en la transacción local. Configuración y operación pendientes. |
| Pasos distribuidos | Saga, cuando el flujo lo requiera | Coordinar estados y compensaciones entre sistemas independientes; no sustituye a Kafka ni a la outbox. |

La comunicación propuesta utiliza una API REST sobre HTTPS con representaciones JSON. REST es independiente del lenguaje y del framework del backend [10]. La autenticación mantiene los protocolos OAuth/OIDC del proveedor.

FastAPI aporta herramientas para esquemas y OpenAPI [1]; SQLAlchemy permite delimitar transacciones y Alembic gestionar migraciones [2][3]. La atomicidad de una transferencia, la idempotencia y la autorización dependen de la implementación del flujo y de sus restricciones.

React Native permite desarrollar para Android e iOS con TypeScript [4][5]. Es una propuesta a evaluar frente al stack que ya mantenga el equipo. Compartir código no elimina las integraciones ni las pruebas por plataforma: la compilación local de iOS requiere Xcode y la de Android su propio entorno [6].

## Almacenamiento seguro de Android

`EncryptedSharedPreferences`, mencionado en el requerimiento inicial, está deprecado en AndroidX Security Crypto 1.1.0 [11]. Queda pendiente seleccionar la implementación de reemplazo.

El criterio propuesto es mantener el access token en memoria y proteger el refresh token persistido con cifrado y claves gestionadas mediante Android Keystore. Keystore almacena claves criptográficas; no equivale a un almacén genérico de tokens [12]. Elegir una integración mantenida y revisar sus dependencias nativas. No guardar tokens en texto claro en preferencias comunes o AsyncStorage [8].

Antes de cerrar la decisión, verificar versiones mínimas de Android, reglas de copia/restauración, pérdida o invalidación de claves, acceso con el dispositivo bloqueado y borrado al cerrar sesión. Estos pendientes se incorporan a autenticación y STRIDE.

## Integración del asistente

El orquestador forma parte del backend modular. El adaptador de Gemini encapsula la llamada externa y conserva una configuración por entorno para modelo, timeout total, límites de salida, cuotas y máximo de funciones por turno. El catálogo inicial admite `get_account_balance` y `prepare_transfer`; la ejecución financiera permanece en el circuito transaccional.

Para Python se propone `google-genai`. El prototipo mostrado utiliza la interfaz heredada `google-generativeai`; migrar implica adaptar cliente, sesiones y manejo de funciones, además de cambiar la dependencia. La disponibilidad de una versión de Gemini debe comprobarse con la configuración real; no se fija un nombre de modelo no verificado en el diseño. [13]

Las credenciales se obtienen mediante la integración del backend con HashiCorp Vault. El archivo de ejemplo de entorno documenta nombres de variables y valores ficticios. El repositorio, la app móvil y los mensajes del modelo no incluyen secretos.

Las primeras pruebas de integración deben separar una respuesta de texto, una consulta real al mock y la preparación de una tarjeta. También deben cubrir respuestas inválidas, fallos de proveedor y que preparar una transferencia no genere movimientos. El contrato objetivo y las diferencias de la demo están en [04-contratos-api.md](04-contratos-api.md).

## Organización propuesta

La organización propuesta mantiene este repositorio para documentación y separa backend y mobile en dos repositorios, cada uno con su configuración y verificaciones. Los cambios de contratos deben coordinarse mediante una versión común de referencia. La organización definitiva queda pendiente.

Las siguientes rutas representan responsabilidades previstas; la estructura definitiva dependerá del stack elegido.

| Proyecto | Área | Responsabilidad |
|---|---|---|
| Backend | `api/` | Rutas, esquemas, respuestas y errores HTTP. |
| Backend | `modules/identity/` | Identidad local, validación de tokens y autorización. |
| Backend | `modules/accounts/` | Consulta de cuentas y saldo autorizado. |
| Backend | `modules/ledger/` | Casos de uso de transferencias, asientos, idempotencia y reversas. |
| Backend | `modules/assistant/` | Conversaciones, funciones permitidas y propuestas para confirmación. |
| Backend | `modules/risk/` | Evaluación de riesgo y política previa a autorizar. |
| Backend | `infrastructure/ai/` | Adaptador de Gemini, esquemas y límites de llamadas. |
| Backend | `infrastructure/secrets/` | Acceso técnico a HashiCorp Vault según permisos. |
| Backend | `infrastructure/events/` | Publicador de outbox y adaptadores de Kafka. |
| Backend | `infrastructure/` | PostgreSQL, configuración e integraciones externas. |
| Backend | `migrations/` y `tests/` | Cambios de esquema y verificaciones. |
| Mobile | `features/` | Pantallas y flujos de sesión, cuentas y chat con tarjetas de confirmación. |
| Mobile | `core/` | Cliente HTTP, integración OIDC y almacenamiento seguro. |
| Mobile | `tests/` | Pruebas del comportamiento de la aplicación. |

El backend se mantiene como una aplicación modular. Las carpetas agrupan responsabilidades dentro de la misma aplicación; las reglas contables se concentran en el circuito del ledger y su transacción SQL.

Una carpeta o módulo no constituye una frontera de seguridad. Si los componentes comparten proceso e identidad técnica, también pueden compartir sus privilegios; el aislamiento efectivo debe verificarse al definir identidades, políticas de Vault y despliegue.

## Alcance del scaffolding

1. Proyecto mínimo que pueda instalarse y compilarse o arrancar localmente, con instrucciones reproducibles y versiones de dependencias fijadas.
2. Configuración de entorno de ejemplo sin secretos y exclusiones de Git adecuadas.
3. Lint y formato: Ruff si se elige Python; lint, formato y comprobación de tipos TypeScript si se adopta React Native [5][7].
4. CI para ejecutar las verificaciones del proyecto en cada pull request. Los checks obligatorios deben existir y haber ejecutado correctamente antes de configurarlos como requeridos.
5. Protección de `main`: revisión por otra persona, checks requeridos, resolución de conversaciones y bloqueo de force push y borrado. Configurar su aplicación a administradores y excepciones según lo acordado; registrar evidencia de las reglas efectivas [9].

La disponibilidad de protección en repositorios privados depende del plan de GitHub y de los permisos de administración [9]. Las reglas deben configurarse y verificarse en el repositorio.

## Seguridad y pendientes

La implementación debe seguir los criterios de [autenticación](03-autenticacion.md), [contratos](04-contratos-api.md) y [STRIDE](05-threat-model.md). La estructura inicial prepara los proyectos; los controles requieren implementación y pruebas. Al implementar el ledger, las pruebas de transacciones y concurrencia deberán usar PostgreSQL aislada, además de las pruebas unitarias.

Quedan pendientes las siguientes definiciones:

- Si la empresa exige un stack concreto para backend o mobile.
- Framework móvil, responsables de ambas plataformas y solución mantenida para proteger tokens en Android.
- Organización y acceso a los repositorios, Mac/Xcode, entorno Android y CI disponibles.
- Proveedor de identidad, Gateway y versiones compatibles de las herramientas elegidas.
- Modelo de Gemini accesible, adaptación al SDK propuesto, límites de uso y tratamiento de datos.
- Despliegue y políticas de HashiCorp Vault; almacenamiento aislado del PII Data Vault.
- Topología, retención y consumidores de Kafka; esquema de outbox y alcance de Saga.

## Referencias oficiales

1. [FastAPI: funcionalidades](https://fastapi.tiangolo.com/features/).
2. [SQLAlchemy: transacciones](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html).
3. [Alembic: migraciones](https://alembic.sqlalchemy.org/en/latest/).
4. [React Native: Android e iOS](https://reactnative.dev/).
5. [React Native: TypeScript](https://reactnative.dev/docs/typescript).
6. [React Native: entorno de desarrollo](https://reactnative.dev/docs/set-up-your-environment).
7. [Ruff](https://docs.astral.sh/ruff/).
8. [React Native: seguridad y almacenamiento](https://reactnative.dev/docs/security).
9. [GitHub: ramas protegidas](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches).
10. [Fielding: estilo arquitectónico REST](https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm).
11. [Android: EncryptedSharedPreferences](https://developer.android.com/reference/androidx/security/crypto/EncryptedSharedPreferences).
12. [Android Keystore](https://developer.android.com/privacy-and-security/keystore).
13. [Google: SDK oficiales y bibliotecas heredadas de Gemini](https://ai.google.dev/gemini-api/docs/libraries).
14. [HashiCorp Vault: KV](https://developer.hashicorp.com/vault/docs/secrets/kv).
15. [HashiCorp Vault: Transit](https://developer.hashicorp.com/vault/docs/secrets/transit).
