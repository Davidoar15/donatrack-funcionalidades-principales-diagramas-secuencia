# Realizar Donación
```mermaid
sequenceDiagram
    actor Cliente
    participant DON as Donaciones
    participant DYE as Donadores y Entidades
    participant LOG as Logística (API)
    participant MQ as RabbitMQ
    participant WRK as Logística (Worker)

    Cliente->>DON: POST /donaciones {donadorID, depositoID, productoID, cantidad}
    activate DON
    DON->>DON: valida cantidad > 0, producto existe localmente

    DON->>DYE: GET /donadores/{donadorID}
    activate DYE
    alt donador no existe
        DYE-->>DON: 404
        deactivate DYE
        DON-->>Cliente: 422 (DonacionRechazadaException)
        deactivate DON
    else donador existe
        DYE-->>DON: 200 DonadorDTO
        deactivate DYE
        DON->>DYE: GET /donadores/{donadorID}/puede-donar
        activate DYE
        DYE-->>DON: true / false
        deactivate DYE
        alt no puede donar
            DON-->>Cliente: 422 (DonacionRechazadaException)
            deactivate DON
        else puede donar
            DON->>DON: crea Donacion (estado=INGRESADA), guarda en BD
            DON->>LOG: POST /depositos/{depositoID}/donacion {donacionID, productoID, cantidad}
            activate LOG
            LOG->>LOG: valida capacidad del depósito,<br/>crea Paquete (PENDIENTE), guarda Depósito
            Note over LOG: @TransactionalEventListener(AFTER_COMMIT):<br/>solo se publica si la transacción de<br/>Logística confirmó
            LOG->>MQ: publica DonacionPendienteMessage<br/>(incluye el traceId del request original)
            LOG-->>DON: 202 Accepted (DepositoDTO)
            deactivate LOG
            DON-->>Cliente: 201 Created (DonacionDTO)
            deactivate DON

            Note over MQ,WRK: --- a partir de acá es asíncrono:<br/>el cliente ya recibió su respuesta ---
            MQ->>WRK: entrega el mensaje
            activate WRK
            WRK->>DYE: GET /necesidades?productoSolicitadoID={productoID}
            activate DYE
            DYE-->>WRK: List de NecesidadMaterialDTO insatisfechas
            deactivate DYE
            loop por cada necesidad candidata
                WRK->>LOG: GET /internal/matchmaking/necesidades/{id}/cantidad-asignada
                activate LOG
                LOG-->>WRK: cantidad ya comprometida
                deactivate LOG
            end
            WRK->>WRK: corre el algoritmo de matchmaking<br/>(FIFO / el configurado en el depósito)
            WRK->>LOG: POST /internal/matchmaking/resultados<br/>{depositoId, paqueteId, necesidadId, cantidadAsignada, cantidadSobrante}
            activate LOG
            alt hay necesidad compatible
                LOG->>LOG: crea Asignacion (ASIGNADA), paquete pasa a ASIGNADO
            else sin necesidad compatible
                LOG->>LOG: paquete pasa a EN_STOCK<br/>(queda disponible para asignación directa,<br/>ver funcionalidad 5)
            end
            LOG-->>WRK: respuesta
            deactivate LOG
            deactivate WRK
        end
    end
```
