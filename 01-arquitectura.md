# 01 · Arquitectura propuesta para la app bancaria

**Estado:** propuesta de arquitectura. La implementación y validación de los controles están pendientes, al igual que la selección de proveedores y la definición de contratos externos.

Propongo una app para Android e iOS, un proveedor de identidad separado y un backend modular. El diseño separa la persistencia y protección de los datos de la integración con eventos y sistemas externos.

## Vista general

![Arquitectura propuesta para la app bancaria](diagrama-arquitectura-bancaria.png)

[Versión vectorial del diagrama](diagrama-arquitectura-bancaria.svg).

El flujo principal sigue el recorrido **app → gateway → backend**. El proveedor de identidad está a la izquierda. Desde el backend se abren las dos ramas inferiores: **persistencia y auditoría**, e **integración externa y resiliencia**. Las notas junto a cada componente indican sus controles principales.

Las flechas muestran relaciones entre componentes o capas; el recorrido concreto depende de la operación. Los recuadros punteados agrupan responsabilidades. Los límites de confianza deben concretarse mediante identidades, permisos y controles de acceso. Los tres módulos del backend forman parte de una misma aplicación.

## Responsabilidades y controles

### Identidad y capa 1 · Cliente / dispositivo

- **Inicio de sesión:** OAuth 2.0 / OIDC con Authorization Code + PKCE. El proveedor emite los tokens; Auth0, Keycloak y Cognito son opciones pendientes de selección.
- **Tokens:** la app presenta el access token a la API. El refresh token permite renovar el acceso según la política del proveedor. El ID token comunica el resultado de la autenticación y no se utiliza como credencial de la API. El access token se mantiene en memoria y el refresh token se protege mediante el almacenamiento seguro definido para cada plataforma.
- **Canal seguro:** HTTPS/TLS y certificate pinning entre la app y el gateway. Deben definirse pins de respaldo, rotación y recuperación. Android desaconseja el pinning de forma general por el riesgo de interrumpir la conectividad ante cambios de certificados; su incorporación necesita una política operativa acordada.
- **Device attestation:** Google Play Integrity en Android y Apple App Attest en iOS. El servidor verifica las evidencias y su vínculo con la solicitud. Aportan señales de integridad de la app y del dispositivo; la identidad del usuario y sus permisos se comprueban por separado.
- **Idempotencia:** la app conserva `X-Idempotency-Key` al reintentar el mismo pedido. El backend persiste la clave, su alcance y contenido para evitar repetir el efecto.
- **Biometría / step-up:** una operación sensible puede requerir verificación adicional. El servidor debe verificar la evidencia y vincularla con la operación concreta.

### Capa 2 · Borde / gateway y seguridad

El **API Gateway / WAF** recibe las solicitudes y concentra los controles de entrada: validación del access token, filtrado de tráfico y rate limiting. Asigna o valida el contexto de seguimiento, incluido `X-Correlation-ID`, para relacionar los registros de una solicitud.

La flecha **JWKS en caché** representa las claves públicas que permiten verificar las firmas de los tokens. Deben comprobarse firma, emisor, destinatario y vigencia, con una política de actualización de claves. El backend mantiene la validación del token y la autorización por recurso, aunque esa dependencia no se repita en el diagrama.

**CORS** aplica a clientes de navegador. Las listas de IP se utilizan en conexiones donde tengan sentido; no se presupone una IP fija para cada teléfono.

### Capa 3 · Dominio / backend modular

| Módulo | Responsabilidad |
| --- | --- |
| **Risk & Anti-Fraud Engine** | Evalúa la operación y las señales verificadas antes de autorizar el movimiento. Puede permitir continuar, rechazar o pedir verificación adicional según la política. |
| **Usuarios y Cuentas** | Comprueba el estado del usuario y su relación con cada cuenta —ownership—, además de los permisos necesarios para operar. |
| **Transaccional / Movimientos** | Coordina estados, idempotencia, fondos y concurrencia. Confirma la operación solo cuando se cumplen todas las reglas. |

La decisión de riesgo debe corresponder al monto, destino y contexto que finalmente se ejecutan. En el flujo interno propuesto, las llamadas externas de evaluación ocurren antes de tomar los bloqueos contables; dentro de la transacción se revalidan fondos y reglas.

La **máquina de estados** permite transiciones justificadas y conserva su historial. Un timeout externo puede dejar el resultado pendiente de confirmar. El estado actual puede cambiar; los asientos confirmados y los hechos históricos se conservan.

### Capa 4 · Persistencia y auditoría

- **Ledger inmutable (Double-Entry):** cada asiento debe balancear débitos y créditos por moneda. Los asientos confirmados se preservan; una corrección agrega una nueva operación vinculada. Reconstruir saldos requiere un ledger completo, incluyendo apertura y contrapartidas.
- **Eventos append-only:** agregan el historial de hechos de cada transacción, con identificadores, tipo, versión y secuencia. Guardar únicamente el estado final no permite reconstruir todo lo sucedido. Adoptar Event Sourcing como fuente de verdad sería una decisión adicional.
- **Outbox transaccional:** en el ejemplo interno, operación, asientos, evento e intención de publicación se guardan en la misma transacción SQL. Los metadatos de entrega pueden actualizarse sin alterar el hecho histórico.
- **PII Data Vault:** protege datos personales sensibles mediante cifrado, acceso restringido y auditoría. El registro contable utiliza referencias y el mínimo de datos necesario. El diagrama agrupa esta responsabilidad en la capa de persistencia. Queda pendiente definir su aislamiento físico y lógico.
- **Secrets Manager / KMS:** se agrupan visualmente para simplificar el esquema, pero tienen funciones distintas: credenciales técnicas y claves criptográficas, respectivamente. Los permisos de lectura y descifrado se restringen por separado. La tecnología concreta queda por seleccionar.

La inmutabilidad del historial necesita controles de escritura, auditoría de accesos privilegiados, conservación de copias y restauración probada. Estas garantías dependen de los controles de almacenamiento y acceso, además del uso de registros append-only.

### Capa 5 · Integración externa y resiliencia

| Componente | Criterio de implementación |
| --- | --- |
| **Event Bus / Message Broker** | Apache Kafka es la opción elegida para la propuesta. Un publicador toma la outbox después del commit y distribuye sus eventos. |
| **Outbox y consumidores** | La publicación admite reintentos y puede producir duplicados. Cada consumidor persiste su deduplicación por `event_id` junto con el efecto local; un efecto externo necesita su propia estrategia idempotente. |
| **Circuit Breakers & Timeouts** | Son controles del adaptador que comunica con el core. El timeout limita la espera; el circuit breaker suspende llamadas al alcanzar la condición de fallos definida. |
| **Core bancario externo** | Se integra mediante un contrato y un adaptador. Deben acordarse identificadores, estados, idempotencia y consulta o conciliación de resultados inciertos. |
| **Saga, si aplica** | Coordina varios pasos en sistemas independientes mediante estados, reintentos y compensaciones. Una compensación financiera genera una nueva operación según las reglas del negocio. |

**Outbox y Kafka se complementan:** la outbox coordina el cambio SQL con la intención de publicar; Kafka distribuye eventos. El publicador y los consumidores están resumidos en las notas del broker para mantener el diagrama compacto.

Kafka utiliza un log append-only, pero la retención, la compactación y las operaciones administrativas pueden eliminar registros. No sustituye al ledger ni a la política de conservación del historial financiero. Deben definirse clave de particionado, orden por entidad, reintentos y tratamiento de mensajes que no pueden procesarse.

**Resultado incierto:** si el core no responde a tiempo, podría haber procesado el pedido. El resultado debe consultarse o conciliarse antes de repetir el movimiento o declararlo fallido. La degradación ofrece funciones limitadas y explícitas, conservando permisos e indicando la antigüedad de datos en caché.

## Flujo de una transferencia interna

El siguiente ejemplo utiliza cuentas ficticias, una misma moneda y operaciones sin comisiones:

1. La app envía la solicitud, el access token y la clave de idempotencia.
2. Gateway y backend validan acceso y permisos. Se comprueban las evidencias requeridas y se evalúa el riesgo antes de autorizar.
3. El backend reconoce los reintentos del mismo pedido. Para una operación nueva, controla concurrencia e idempotencia dentro de la transacción, bloquea lo necesario y revalida fondos y reglas.
4. Una sola transacción SQL registra la operación, la partida doble, el evento y la outbox. Se responde con éxito después del commit; un fallo previo revierte el conjunto.
5. El publicador envía el evento confirmado a Kafka. Los consumidores procesan sus efectos sin duplicarlos.

Si la respuesta al cliente se pierde, reintentar con la misma clave y contenido recupera la operación existente. Reutilizar la clave con otro contenido se rechaza. La unicidad debe resolver también solicitudes concurrentes, verificando permisos en cada intento.

Este ejemplo supone que PostgreSQL registra ambos lados de la transferencia interna. **Queda pendiente definir qué sistema será el registro oficial de los movimientos cuando intervenga el core.** Una transacción SQL local no abarca automáticamente al core ni al Data Vault. Saga solo se incorpora cuando exista un flujo distribuido que la necesite.

## Controles transversales y definiciones pendientes

Las trazas, logs, métricas y alertas permiten seguir solicitudes, fallos y eventos pendientes. La auditoría conserva los hechos relevantes. Los registros técnicos evitan contraseñas, tokens y datos personales sensibles.

El modelo de amenazas **STRIDE** acompaña el diseño: suplantación, manipulación, repudio, exposición de información, denegación de servicio y elevación de privilegios. Cada control necesita un escenario y una verificación en `05-threat-model.md`.

Quedan pendientes las siguientes definiciones:

1. **Ledger y core:** fuente oficial del dinero, estados, evidencia de confirmación y conciliación.
2. **Riesgo y autenticación adicional:** reglas, vigencia y comportamiento ante indisponibilidad o dispositivos sin soporte.
3. **Datos y secretos:** aislamiento del Data Vault, permisos de descifrado, proveedores y operación del pinning.
4. **Eventos y operaciones distribuidas:** retención, orden, reintentos, límites de acumulación y alcance de Saga.

## Referencias

- [AWS: Transactional Outbox](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html), [Saga](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html) y [reintentos idempotentes](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/).
- [Apache Kafka: diseño](https://kafka.apache.org/42/design/design/) y [retención y compactación](https://kafka.apache.org/43/configuration/topic-configs/).
- [OWASP: autorización de transacciones](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html) y [seguridad REST](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html).
- [Android: Play Integrity](https://developer.android.com/google/play/integrity/standard) y [seguridad de red y pinning](https://developer.android.com/privacy-and-security/security-ssl).
- [Apple: App Attest](https://developer.apple.com/videos/play/wwdc2021/10244/) y [AWS: conceptos de KMS](https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html).
- [Microsoft: Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) y [OpenTelemetry: contexto de trazas](https://opentelemetry.io/docs/concepts/context-propagation/).

<details>
<summary>Diagrama en Mermaid</summary>

```mermaid
flowchart TB
    IDP["`Proveedor de identidad (IdP)
Auth0 / Keycloak / Cognito
Emisión de tokens`"]

    subgraph CLIENTE["1. Capa cliente / dispositivo"]
        APP["`App móvil (iOS / Android)
Pinning · Device Attestation
X-Idempotency-Key · biometría / step-up`"]
    end

    subgraph BORDE["2. Capa de borde / gateway y seguridad"]
        GW["`API Gateway / WAF
X-Correlation-ID · rate limiting
JWT · CORS / IP allowlist según canal`"]
    end

    subgraph BE["3. Capa de dominio / backend modular"]
        direction LR
        RISK["`Risk & Anti-Fraud Engine
Antes de autorizar`"]
        CUENTAS["`Usuarios y Cuentas
Ownership Validation`"]
        TX["`Transaccional / Movimientos
Estados · idempotencia · concurrencia`"]
    end

    subgraph DATOS["4. Capa de persistencia y auditoría"]
        direction LR
        PG["`PostgreSQL
Ledger inmutable (Double-Entry)
Eventos append-only · outbox
PII Data Vault cifrado y aislado`"]
        KEYS["`Secrets Manager / KMS
Claves y credenciales
Permisos separados`"]
        PG <--> KEYS
    end

    subgraph INTEGRACION["5. Capa de integración externa y resiliencia"]
        direction LR
        BROKER["`Event Bus / Message Broker
Kafka · Outbox Pattern
Consumidores idempotentes
Publicación tras el commit`"]
        RES["`Circuit Breakers & Timeouts
Consulta y conciliación
Saga si hay pasos distribuidos`"]
        CORE["`Core bancario externo
Vía adaptador`"]
        RES <--> CORE
    end

    APP <-->|"OAuth 2.0 / OIDC: Code + PKCE"| IDP
    IDP -.->|"JWKS en caché"| GW
    APP -->|"HTTPS / TLS · access token"| GW
    GW -->|"Solicitud a la API"| BE
    BE --> DATOS
    BE --> INTEGRACION

    classDef componente fill:#202524,stroke:#979b9f,color:#efefef
    class APP,IDP,GW,RISK,CUENTAS,TX,PG,KEYS,BROKER,RES,CORE componente
    style CLIENTE fill:#101318,stroke:#85898e,color:#efefef,stroke-dasharray:3 5
    style BORDE fill:#101318,stroke:#85898e,color:#efefef,stroke-dasharray:3 5
    style BE fill:#101318,stroke:#85898e,color:#efefef,stroke-dasharray:3 5
    style DATOS fill:#101318,stroke:#85898e,color:#efefef,stroke-dasharray:3 5
    style INTEGRACION fill:#101318,stroke:#85898e,color:#efefef,stroke-dasharray:3 5
    linkStyle default stroke:#979b9f,color:#efefef
```

El PNG y el SVG conservan la composición del diagrama. La versión Mermaid contiene los mismos componentes con distribución automática.

</details>
