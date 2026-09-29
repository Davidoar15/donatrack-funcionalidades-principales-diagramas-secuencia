# Realizar Donación
```mermaid
sequenceDiagram
    actor Cliente
    participant DON as Donaciones
    participant DYE as Donadores y Entidades
    participant LOG as Logística (API)
    participant MQ as RabbitMQ
    participant WRK as Logística (Worker)

    Cliente->>DON: POST /donaciones
    activate DON
    DON->>DON: valida cantidad > 0

    DON->>DYE: GET /donadores/{donadorID}
    alt donador no existe
        DYE-->>DON: 404
        DON-->>Cliente: 422
        deactivate DON
    else donador existe
        DYE-->>DON: 200 DonadorDTO
        DON->>DYE: GET /donadores/{id}/puede-donar
        alt no puede donar
            DON-->>Cliente: 422
            deactivate DON
        else puede donar
            DON->>DON: crea Donacion
            DON->>LOG: POST /depositos/{id}/donacion
            activate LOG
            LOG->>LOG: valida capacidad,<br/>crea Paquete
            LOG->>MQ: publica evento
            LOG-->>DON: 202 Accepted
            deactivate LOG
            DON-->>Cliente: 201 Created
            deactivate DON

            Note over MQ,WRK: --- asíncrono (cliente ya recibió) ---
            MQ->>WRK: entrega mensaje
            activate WRK
            WRK->>DYE: GET /necesidades
            DYE-->>WRK: List necesidades
            loop por cada necesidad
                WRK->>LOG: GET cantidad-asignada
                LOG-->>WRK: cantidad
            end
            WRK->>WRK: ejecuta matchmaking
            WRK->>LOG: POST /resultados
            activate LOG
            alt hay compatibilidad
                LOG->>LOG: crea Asignacion
            else sin compatibilidad
                LOG->>LOG: paquete → EN_STOCK
            end
            deactivate LOG
            deactivate WRK
        end
    end
```
