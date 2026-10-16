# 06 · Stack y estructura inicial

**Confirmado en el alcance:** Android e iOS; PostgreSQL está indicado en el requerimiento. Gemini, HashiCorp Vault y Kafka forman parte de las decisiones documentadas. Frameworks, configuración operativa, repositorios y CI siguen pendientes.

La elección final debe considerar las herramientas que ya mantiene el equipo, las integraciones nativas y la capacidad de aplicar las garantías del ledger.

## 1. Tecnologías propuestas

| Componente | Propuesta | Motivo y estado |
|---|---|---|
| Comunicación | REST sobre HTTPS y JSON | Contrato común para ambas plataformas, alineado con [04](04-contratos-api.md). |
| Base de datos | PostgreSQL | Requerido. Aporta transacciones y restricciones; falta definir su autoridad frente al core. |
| Backend | Python + FastAPI | Candidato para la aplicación modular; facilita validación de esquemas y OpenAPI. Sujeto al estándar del equipo. |
| Persistencia | SQLAlchemy + Alembic | Candidatos si se adopta Python: acceso a datos, transacciones explícitas y migraciones versionadas. |
| Mobile | React Native + TypeScript | Candidato para compartir código; validar experiencia, SDK de identidad e integraciones en ambas plataformas. |
| Sesión local | Keychain / cifrado con Keystore | Criterios por plataforma; biblioteca Android pendiente. Ver [autenticación](03-autenticacion.md#4-renovación-y-almacenamiento). |
| Identidad y gateway | Selección pendiente | Evaluar primero soluciones de la empresa, manteniendo OAuth/OIDC y autorización en backend. |
| IA | Gemini; `google-genai` para Python | Servicio elegido y SDK oficial propuesto. Modelo, cuotas y límites se verifican por entorno. |
| Secretos y claves | HashiCorp Vault | Políticas por servicio y secreto; evaluar Transit para cifrado. Configuración y recuperación pendientes. |
| Eventos | Kafka + outbox | Publicación posterior al commit y consumidores idempotentes. Falta cerrar esquema y operación. |
| Pasos distribuidos | Saga cuando se necesite | Coordina estados y compensaciones entre sistemas independientes. |

REST describe el estilo de la interfaz; FastAPI es una herramienta para implementarla. El framework y el ORM no garantizan por sí mismos atomicidad, idempotencia o autorización: esas reglas se implementan en los casos de uso y en la base.

Compartir código móvil mantiene la necesidad de pruebas nativas. La compilación local de iOS requiere Mac/Xcode y Android su propio entorno. `EncryptedSharedPreferences` está deprecado; el criterio de reemplazo y las comprobaciones se concentran en autenticación.

## 2. Organización del código

Se propone mantener este repositorio como documentación y separar backend y mobile, coordinando una versión común de los contratos. La organización definitiva queda pendiente. Las rutas siguientes agrupan responsabilidades dentro de cada proyecto:

| Proyecto / ruta | Responsabilidad |
|---|---|
| Backend · `api/` | Rutas, esquemas, respuestas y errores HTTP. |
| Backend · `modules/identity/` | Identidad local, tokens y autorización. |
| Backend · `modules/accounts/` | Cuentas y saldo autorizado. |
| Backend · `modules/ledger/` | Transferencias, asientos, idempotencia y reversiones. |
| Backend · `modules/risk/` | Evaluación previa a autorizar. |
| Backend · `modules/assistant/` | Conversaciones, funciones permitidas y propuestas. |
| Backend · `infrastructure/` | PostgreSQL y adaptadores externos; `ai/`, `secrets/` y `events/` para Gemini, Vault y outbox/Kafka. |
| Backend · `migrations/`, `tests/` | Evolución del esquema y verificaciones. |
| Mobile · `features/` | Flujos de sesión, cuentas, chat y tarjetas de confirmación. |
| Mobile · `core/`, `tests/` | Cliente HTTP, OIDC, almacenamiento seguro y pruebas. |

El backend sigue siendo una aplicación modular. **Las carpetas organizan código; el aislamiento de permisos depende del proceso, sus credenciales y el despliegue.** Las reglas contables se concentran en el circuito del ledger.

### Adaptador de Gemini

Encapsula la llamada externa y configura modelo, tiempo total, salida, cuotas y máximo de funciones por turno. El catálogo admite `get_account_balance` y `prepare_transfer`; la ejecución conserva el circuito transaccional y la confirmación del usuario.

Se propone el SDK mantenido `google-genai`. Cualquier código heredado con `google-generativeai` requiere adaptar cliente, sesiones y funciones. La credencial se obtiene mediante Vault; el ejemplo de entorno documenta nombres y valores ficticios, nunca claves reales.

Las primeras verificaciones separan texto, consulta al mock y preparación de tarjeta. Deben incluir fallos del proveedor, respuestas inválidas y ausencia de movimientos al preparar. El diseño del contrato está en [04](04-contratos-api.md#5-asistente).

## 3. Scaffolding: punto de partida ejecutable

El scaffolding es la estructura mínima para que el equipo pueda instalar, ejecutar y verificar los proyectos de forma consistente:

1. Arranque o compilación local con instrucciones reproducibles y dependencias fijadas.
2. Configuración de ejemplo sin secretos y exclusiones de Git.
3. Lint y formato; Ruff si se elige Python, más comprobación de tipos si se adopta TypeScript.
4. CI con verificaciones en cada pull request. Los checks deben existir y pasar antes de exigirlos.
5. Protección de `main`: revisión de otra persona, checks y conversaciones resueltas, sin force push ni borrado. Verificar excepciones, permisos y disponibilidad según el plan.

Al implementar el ledger, las pruebas de transacciones y concurrencia necesitan PostgreSQL aislado además de pruebas unitarias. La estructura inicial no acredita que los controles de [STRIDE](05-threat-model.md) ya funcionen.

## 4. Pendientes

Acordar backend y framework móvil; organización y acceso a repositorios; entornos de Android/iOS y CI; IdP/Gateway; almacenamiento Android; versiones compatibles. Para Gemini, Vault y Kafka faltan modelo/configuración, políticas de datos y permisos, límites, retención y recuperación. Los detalles funcionales se mantienen en sus secciones correspondientes.

## Para profundizar

- [FastAPI](https://fastapi.tiangolo.com/features/), [SQLAlchemy: transacciones](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html) y [Alembic](https://alembic.sqlalchemy.org/en/latest/).
- React Native: [entorno de desarrollo](https://reactnative.dev/docs/set-up-your-environment) y [TypeScript](https://reactnative.dev/docs/typescript).
- [Google: bibliotecas oficiales de Gemini](https://ai.google.dev/gemini-api/docs/libraries).
- [GitHub: ramas protegidas](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches).
