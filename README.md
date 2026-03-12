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
    %% Representación de Actor (Monigote) en lugar de simple participante
    actor S as "Socio"
    participant IW as "InterfazWeb"
    participant GM as "BookingManager"
    participant DB as "Database"

    Note over S, DB: El flujo es cronológico (de arriba hacia abajo)

    %% Mensaje Síncrono (Flecha rellena): El emisor espera respuesta
    S->>IW: confirmarReserva()
    activate S
    activate IW
    Note right of S: El socio inicia la acción. La activación indica el foco de control.

    IW->>GM: confirmBooking(memberId, classId)
    activate GM
    
    GM->>DB: checkAvailability(classId)
    activate DB
    %% Mensaje de Retorno (Línea discontinua): Devuelve el valor solicitado
    DB-->>GM: availabilityStatus
    deactivate DB
    Note right of DB: Retorno con el estado de ocupación

    alt Is Available
        Note over GM, DB: Caso: Hay plazas disponibles (Fragmento Combinado Alt)
        GM->>DB: createBooking(memberId, classId)
        activate DB
        DB-->>GM: success
        deactivate DB
        
        GM-->>IW: bookingConfirmed
        deactivate GM
        
        IW-->>S: Mostrar mensaje de éxito
        deactivate IW
        deactivate S
    else Is Full
        Note over GM, DB: Caso: La clase está llena
        GM-->>IW: bookingFailed(Full)
        activate GM
        deactivate GM
        
        IW-->>S: Mostrar mensaje de clase llena/Lista de espera
        activate IW
        deactivate IW
    end
```

### Tarea 3: Diagrama de Comunicación

Este diagrama muestra la misma interacción pero enfocada en las relaciones de los objetos y el orden de los mensajes.

```mermaid
graph LR
    %% Orquestación de objetos para procesar la reserva
    S[:Socio] -- "1: confirmReservation()" --> IW[:InterfazWeb]
    IW -- "2: processBooking(mId, cId)" --> GM[:BookingManager]
    GM -- "3: checkCapacity(cId)" --> DB[:Database]
    DB -- "3.1: capacityResponse" --> GM
    %% Rama condicional si el aforo es suficiente
    GM -- "4 [if OK]: createBooking(mId, cId)" --> DB
    GM -- "5: notifyResult(status)" --> IW
    IW -- "5.1: showMessage()" --> S
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
