# # Documentación Técnica: Módulo de Gestión de Clases Colectivas - GymMaster

Este documento contiene el análisis y diseño técnico del sistema de gestión de reservas para el gimnasio **GymMaster**.

---

## Fase 1: Análisis de Requisitos

### Tarea 1: Diagrama de Casos de Uso

El siguiente diagrama describe las interacciones entre los actores y el sistema.

- **Actores**: `Member` (Socio) y `Admin` (Administrador).
- **Incluye**: Se requiere el `Login` para realizar cualquier acción crítica.
- **Extiende**: La opción de `Join Waiting List` aparece solo si la clase está llena.

```mermaid

graph LR
    subgraph SystemBoundary ["Gimnasio GymMaster"]
        UC_Login(Login)
        UC_Reserve(Reserve Class)
        UC_Waitlist(Join Waiting List)
        UC_ManageClasses(Manage Classes)
        UC_CancelSession(Cancel Session)

        UC_Reserve -.-> |"<<include>>"| UC_Login
        UC_Waitlist -.-> |"<<extend>>"| UC_Reserve
        UC_ManageClasses -.-> |"<<include>>"| UC_Login
        UC_CancelSession -.-> |"<<include>>"| UC_Login
    end

    ActorMember((Socio))
    ActorAdmin((Administrador))

    ActorMember --> UC_Reserve
    ActorAdmin --> UC_ManageClasses
    ActorAdmin --> UC_CancelSession
```

## Fase 2: Diseño de la Interacción

### Tarea 2: Diagrama de Secuencia "Confirmar Reserva"

Representa el flujo temporal desde que el Socio pulsa el botón de confirmar.

sequenceDiagram
    autonumber
    
    %% Definición de participantes y actores
    actor S as Socio
    participant IW as InterfazWeb
    participant GM as BookingManager
    participant DB as Database

    %% Flujo de mensajes
    S->>IW: confirmarReserva()
    activate S
    activate IW
    Note right of S: El socio pulsa confirmar
    
    IW->>GM: confirmBooking(memberId, classId)
    activate GM
    
    GM->>DB: checkAvailability(classId)
    activate DB
    DB-->>GM: availabilityStatus
    deactivate DB
    Note right of DB: Respuesta de disponibilidad

    alt Hay plazas (Is Available)
        GM->>DB: createBooking(memberId, classId)
        activate DB
        DB-->>GM: success
        deactivate DB
        
        GM-->>IW: bookingConfirmed
        IW-->>S: Mostrar éxito
    else Clase llena (Is Full)
        GM-->>IW: bookingFailed(Full)
        IW-->>S: Mostrar error / Lista espera
    end

    deactivate GM
    deactivate IW
    deactivate S
```

### Tarea 3: Diagrama de Comunicación

Este diagrama muestra la misma interacción pero enfocada en las relaciones de los objetos y el orden de los mensajes.

flowchart LR
    %% Definición de los objetos involucrados
    O1["Socio"]
    O2["InterfazWeb"]
    O3["BookingManager"]
    O4["Database"]

    %% Enlaces con mensajes numerados
    O1 -- "1: confirmReservation()" --> O2
    O2 -- "2: processBooking()" --> O3
    O3 -- "3: checkCapacity()" --> O4
    O4 -. "3.1: status" .-> O3
    O3 -- "4 [if OK]: createBooking()" --> O4
    O3 -. "5: notifyResult()" .-> O2
    O2 -. "5.1: showMessage()" .-> O1
    end
    ```

## Fase 3: Lógica del Proceso

### Tarea 4: Diagrama de Actividades "Validación de Reserva"

Muestra el flujo lógico interno antes de consolidar la reserva.

```mermaid
flowchart TD
    %% Flujo lógico de validación de negocio
    Start((Inicio)) --> Rec["1. Recibir solicitud<br/>(Receive Request)"]
    Rec --> DecPaid{"¿Socio tiene<br/>cuota pagada?<br/>(Has Paid Fee?)"}
  
    DecPaid -- No --> EndError((Fin - Error Pago))
    DecPaid -- Sí --> DecCap{"¿Hay aforo?<br/>(Is Capacity Available?)"}
  
    %% Gestión de excepciones de aforo
    DecCap -- No --> EndFull((Fin - Sugerir<br/>Lista Espera))
    DecCap -- Sí --> Block["4. Bloquear plaza<br/>(Block Spot)"]
  
    Block --> Email["5. Enviar email de<br/>confirmación<br/>(Send Confirmation Email)"]
    Email --> End((Fin))
    end
```

## Fase 4: Ciclo de Vida del Objeto

### Tarea 5: Diagrama de Estados del Objeto "Reserva" (Booking)

Define los estados por los que pasa una reserva y los eventos que disparan la transición.

```mermaid
stateDiagram-v2
    %% Ciclo de vida: de la creación al cierre
    [*] --> Pending : requestBooking()
  
    Pending --> Confirmed : confirm()
    Pending --> Cancelled : cancel()
  
    %% Transiciones después de la confirmación
    Confirmed --> Attended : checkIn()
    Confirmed --> Cancelled : cancel()
    Confirmed --> NoShow : gracePeriodExpired()
  
    %% Estados finales
    Attended --> [*]
    Cancelled --> [*]
    NoShow --> [*]
```
