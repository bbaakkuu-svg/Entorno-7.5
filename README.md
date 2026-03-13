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

        UC_Reserve -.-> |&lt;&lt;include&gt;&gt;| UC_Login
        UC_Waitlist -.-> |&lt;&lt;extend&gt;&gt;| UC_Reserve
        UC_ManageClasses -.-> |&lt;&lt;include&gt;&gt;| UC_Login
        UC_CancelSession -.-> |&lt;&lt;include&gt;&gt;| UC_Login
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
 ```mermaid
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

```mermaid
graph LR

%% Definición de los objetos
Member((:Member))
WebInterface((:WebInterface))
ReservationManager((:ReservationManager))
Database((:Database))

%% --- FLUJO PRINCIPAL ---
Member
-- "1: confirmReservation()"
--> WebInterface

WebInterface
-- "1.1: processReservation(memberId, classId)"
--> ReservationManager

ReservationManager
-- "1.1.1: checkAvailability(classId)"
--> Database

Database
-- "1.1.2: availabilityStatus"
--> ReservationManager


%% --- ALTERNATIVA A: Hay hueco (isAvailable=true) ---
ReservationManager
-- "1.1.3a: [isAvailable=true] blockSpot(memberId, classId)"
--> Database

Database
-- "1.1.4a: [isAvailable=true] successConfirmation"
--> ReservationManager

ReservationManager
-- "1.1.5a: [isAvailable=true] notifySuccess()"
--> WebInterface

WebInterface
-- "1.1.6a: [isAvailable=true] showSuccessMessage()"
--> Member


%% --- ALTERNATIVA B: Clase llena (isAvailable=false) ---
ReservationManager
-- "1.1.3b: [isAvailable=false] notifyFullCapacity()"
--> WebInterface

WebInterface
-- "1.1.4b: [isAvailable=false] showWaitlistOption()"
--> Member
```

---


## Fase 3: Lógica del Proceso

### Tarea 4: Diagrama de Actividades "Validación de Reserva"

Muestra el flujo lógico interno antes de consolidar la reserva.
* **Pasos:** 1. Recibir solicitud -> 2. ¿Socio tiene cuota pagada? (Decisión) -> 3. ¿Hay aforo? (Decisión) -> 4. Bloquear plaza -> 5. Enviar email de confirmación.
* Usa correctamente los símbolos de inicio, fin, acciones y rombos de decisión.
* 
```mermaid
stateDiagram-v2
    
%% Decisiones
state if_fee <<choice>>
state if_capacity <<choice>>

%% Flujo
[*] --> ReceiveRequest
ReceiveRequest --> if_fee
    
%% Primera Decisión: Cuota
if_fee --> if_capacity : [fee paid]
if_fee --> RejectUnpaid : [unpaid fee]
    
%% Segunda Decisión: Aforo
if_capacity --> BlockSpot : [capacity]
if_capacity --> RejectFull : [no capacity]
    
%% Finalización exitosa
BlockSpot --> SendConfirmationEmail
SendConfirmationEmail --> [*]
```

---

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

