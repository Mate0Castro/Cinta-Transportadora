# Diagrama de FLUX


flowchart TD
    A[Inicio] --> B[Inicializar NVS]
    B --> C[Iniciar WiFi]
    C --> D[Iniciar MQTT]
    D --> E[Configurar GPIOs]
    E --> F[Encender Láseres]
    F --> G[Crear Tasks]

    G -->|Task 1| H[Leer Sensores]
    G -->|Task 2| I[Control Motores]
    G -->|Task 3| J[MQTT Task]

    subgraph "Task Sensores"
        H --> K{¿Botón Emergencia?}
        K -->|Sí| L[Detener Sistema]
        K -->|No| M{Sistema Activo?}
        M -->|Sí| N[Leer Fotoresistencias]
        N --> O{¿Nivel < Umbral?}
        O -->|Sí| P[Enviar Mensaje MQTT]
        O -->|No| M
        L --> K
    end

    subgraph "Task Motores"
        I --> Q{Sistema Activo?}
        Q -->|Sí| R[Mover Motor 1]
        R --> S[Mover Motor 2]
        S --> Q
        Q -->|No| T[Detener Motores]
        T --> Q
    end

    subgraph "Task MQTT"
        J --> U[Iniciar Cliente MQTT]
        U --> V[Esperar 10s]
        V --> J
    end
