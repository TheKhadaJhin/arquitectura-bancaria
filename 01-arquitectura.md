```mermaid
flowchart TB
    subgraph CLIENTE["Dispositivo del cliente"]
        APP["App móvil"]
    end

    subgraph IDENTIDAD["Dominio de identidad"]
        IDP["Proveedor de identidad"]
    end

    subgraph SERVICIOS["Servicios de la aplicación"]
        GATEWAY["API Gateway"]
        BACKEND["Backend modular"]
    end

    subgraph DATOS["Acceso restringido a datos"]
        DB[("PostgreSQL")]
    end

    APP <-->|"OAuth 2.0 / OIDC: Authorization Code + PKCE"| IDP

    APP -->|"API REST por HTTPS con access token"| GATEWAY
    GATEWAY -->|"Solicitud a la API"| BACKEND
    BACKEND -->|"Consultas y transacciones SQL"| DB

    BACKEND -.->|"Configuración y claves públicas de validación"| IDP
```


Propongo organizar la aplicación con un proveedor de identidad separado y un backend modular. El gateway concentra los controles generales de entrada; el backend valida el access token, los permisos sobre cada recurso y las reglas de negocio. PostgreSQL almacena los datos y permite agrupar las escrituras en transacciones.
Los módulos de usuarios, cuentas y movimientos forman parte de una misma aplicación backend. Los recuadros delimitan las zonas de confianza y ayudan a identificar dónde aplicar controles.
La línea punteada representa la obtención de configuración y claves públicas del proveedor de identidad. Esa dependencia no implica consultar al proveedor en cada petición.
Esta propuesta supone que PostgreSQL registra los movimientos de dinero. La integración con un core bancario externo queda pendiente de confirmar con el equipo.