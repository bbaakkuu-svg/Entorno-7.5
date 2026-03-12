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
    subgraph SystemBoundary [Gimnasio GymMaster]
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
```Entorno-7.5