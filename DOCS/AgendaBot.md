# Documentación Técnica — AgendaBot

**Proyecto:** AgendaBot Services
**Autor:** Jhoan Sebastian Buitrago Sandoval
**Programa:** Técnico en Desarrollo de Software — Campuslads
**Stack:** Telegram + n8n (Community Edition) + Google Sheets

---

## Tabla de contenido

1. [Introducción y objetivo](#1-introducción-y-objetivo)
2. [Arquitectura general](#2-arquitectura-general)
3. [Modelo de datos (Google Sheets)](#3-modelo-de-datos-google-sheets)
4. [Enrutamiento y navegación](#4-enrutamiento-y-navegación)
5. [Módulos funcionales](#5-módulos-funcionales)
6. [Flujo guiado de agendamiento (Wizard)](#6-flujo-guiado-de-agendamiento-wizard)
7. [Automatizaciones programadas](#7-automatizaciones-programadas)
8. [Validaciones implementadas](#8-validaciones-implementadas)
9. [Manejo de errores](#9-manejo-de-errores)
10. [Registro de usuarios y control de permisos](#10-registro-de-usuarios-y-control-de-permisos)
11. [Logging y trazabilidad](#11-logging-y-trazabilidad)
12. [Limitaciones conocidas](#12-limitaciones-conocidas)
13. [Plan de pruebas](#13-plan-de-pruebas)

---

## 1. Introducción y objetivo

AgendaBot es un bot conversacional que permite agendar, planificar y automatizar tareas personales básicas a través de Telegram, sin depender de plataformas de pago ni servicios que exijan tarjeta de crédito. Todo el enrutamiento, la lógica de negocio y la persistencia de datos viven dentro de un único workflow de n8n (`Agente_Bot_11`), respaldado por una hoja de cálculo de Google Sheets (`AgendaBot_DB`) que actúa como base de datos relacional simplificada.

El diseño sigue cinco principios conversacionales fijos:

- El usuario siempre elige qué hacer escribiendo un número.
- El bot siempre explica qué hace y qué opciones hay.
- El bot siempre sugiere una opción recomendada.
- El bot nunca asume intención.
- El bot siempre ofrece una salida (volver o cancelar, normalmente con "9").

---

## 2. Arquitectura general

```
Telegram Trigger
      │
      ▼
Normalizar sesión ──► lee SESIONES + USUARIOS, determina
      │                pantalla_actual, paso_actual y permisos
      ▼
¿Usuario existe / tiene permiso?
      │
      ├── No existe ──► Registrar usuario nuevo ──► Menú principal
      ├── Existe, sin permiso ──► Acceso restringido
      └── Existe, con permiso ──► Switch (router principal)
                                        │
        ┌───────────┬───────────┬──────┼──────┬───────────┬─────────────┐
        ▼           ▼           ▼      ▼      ▼           ▼             ▼
   Menú principal  Agenda    Tareas  Hábitos Listas    Informes    Configuración
   (Switch1)     (Switch      │        │       │           │       (en construcción)
                  Agenda +   Cerebro  Cerebro  Cerebro    Cerebro
                  Cerebro    Tareas   Habitos  Listas     Informes
                  Agenda)
```

Cada módulo funcional sigue el mismo patrón de arquitectura, replicado de forma consistente en todo el workflow:

```
Get [Hoja(s)] → Cerebro [Módulo] (Code node) → Enviar respuesta [módulo] (Telegram)
                        │                              │
                        ▼                              ▼
              Accion [Módulo] (Switch)         Actualizar SESIONES [módulo]
                        │                              │
              Guardar/Actualizar en Sheets       Log REGISTROS [módulo]
```

El nodo `Cerebro [Módulo]` es siempre un **Code node (JavaScript)** que concentra toda la lógica conversacional de ese módulo: interpreta el texto recibido, valida según el `paso_actual` guardado en sesión, decide la respuesta y calcula qué se debe escribir en las hojas correspondientes.

---

## 3. Modelo de datos (Google Sheets)

Documento: **AgendaBot_DB**

| Hoja | Columnas | Propósito |
|---|---|---|
| **CITAS** | id_cita, fecha, hora, nombre, motivo, canal, estado, creado_por, timestamp_creacion | Citas agendadas y su ciclo de vida (pendiente → cancelada / completada) |
| **TAREAS** | id_tarea, titulo, prioridad, estado, fecha_objetivo, creado_por | Tareas del usuario |
| **HABITOS** | id_habito, nombre, frecuencia, hora_recordatorio, estado | Hábitos y su horario de recordatorio |
| **LISTAS** | id_lista, nombre_lista, tipo, creado_por | Listas (compras, pendientes, ideas) |
| **ITEMS_LISTA** | id_item, id_lista, item, estado | Ítems dentro de cada lista |
| **USUARIOS** | telegram_user, nombre, rol, permitido | Control de acceso y roles (admin / usuario) |
| **LOGS** | timestamp, telegram_user, pantalla, opcion_elegida, resultado | Trazabilidad de toda interacción |
| **SESIONES** | telegram_user, pantalla_actual, paso_actual, datos_parciales, timestamp_ultima_interaccion | Estado conversacional de cada usuario |

**Convenciones de datos:**
- Los booleanos se representan como texto: `"si"` / `"no"` (no `TRUE`/`FALSE`).
- `datos_parciales` almacena JSON serializado como string, usado para mantener información entre pasos de un flujo guiado (por ejemplo, la fecha ya capturada mientras se pide la hora).
- Los IDs se generan con el patrón `PREFIJO-timestamp` (ej. `CITA-20260801143000`), garantizando unicidad sin necesidad de autoincrementales.

---

## 4. Enrutamiento y navegación

El enrutamiento ocurre en dos niveles:

1. **Router principal (`Switch`):** decide a qué módulo va el mensaje según `pantalla_actual` guardado en SESIONES (`menu_principal`, `agenda`, `agenda_nueva_cita`, `tareas`, `habitos`, `listas`, `ayuda`, `recordatorios`, `informes`, `configuracion`).
2. **Sub-routers por módulo:** algunos módulos (como Agenda) tienen un Switch adicional para sus opciones internas (1–5, con salida de fallback para continuar flujos de varios pasos).

Cuando un módulo necesita una conversación de varios pasos (por ejemplo, cancelar una cita: pedir ID → confirmar), el estado del paso se guarda en `paso_actual` dentro de SESIONES, y el propio `Cerebro [Módulo]` interpreta ese paso en el siguiente mensaje — sin necesidad de registrar cada paso intermedio como una pantalla nueva en el router principal.

Toda pantalla ofrece la opción **"9. Volver al menú principal / Cancelar"**, disponible en cualquier punto del flujo.

---

## 5. Módulos funcionales

### 5.1 Agenda
Permite: agendar nueva cita (wizard de 6 pasos), consultar agenda, reprogramar, cancelar y marcar como completada. Ver detalle del wizard en la [sección 6](#6-flujo-guiado-de-agendamiento-wizard).

### 5.2 Tareas
CRUD completo con estados (pendiente / en progreso / completada) y prioridad (alta / media / baja).

### 5.3 Hábitos
Registro de hábitos con frecuencia y hora de recordatorio. Vinculado a la automatización de recordatorios por hora (ver [sección 7](#7-automatizaciones-programadas)).

### 5.4 Listas
Creación de listas (compras, pendientes, ideas), consulta, agregar ítems, marcar ítems como completados y eliminar listas.

### 5.5 Informes
Consulta consolidada de citas próximas, tareas pendientes y hábitos activos.

### 5.6 Configuración
🟡 **Pendiente.** Actualmente muestra un mensaje informativo. Ver [Limitaciones conocidas](#12-limitaciones-conocidas).

---

## 6. Flujo guiado de agendamiento (Wizard)

El flujo de "Agendar nueva cita" es el más elaborado del proyecto — 6 pasos secuenciales, cada uno validado antes de avanzar:

| Paso | Pide | Validación |
|---|---|---|
| 1 | Fecha (`YYYY-MM-DD`) | Formato correcto + no puede ser fecha pasada |
| 2 | Hora (`HH:MM`, 24h) | Formato correcto |
| 3 | Nombre del cliente | No vacío |
| 4 | Motivo | No vacío |
| 5 | Canal (presencial / virtual / llamada) | Opción válida del listado |
| 6 | Confirmación | El usuario revisa el resumen completo antes de guardar |

Antes de confirmar, el sistema verifica que no exista ya otra cita del mismo usuario en la misma fecha y hora (**anti-doble-reserva**). Cancelar, reprogramar y completar reutilizan este mismo patrón de selección + confirmación, y reprogramar reutiliza las mismas validaciones de fecha/hora del wizard original.

---

## 7. Automatizaciones programadas

| Automatización | Disparador | Descripción |
|---|---|---|
| **Resumen diario** | Schedule Trigger, 7:00 a.m. | Envía a cada usuario permitido un resumen de sus citas próximas y tareas pendientes |
| **Recordatorio de hábitos** | Schedule Trigger, cada hora | Revisa qué hábitos tienen `hora_recordatorio` igual a la hora actual y envía un aviso por Telegram |

---

## 8. Validaciones implementadas

- ✅ Opción de menú válida según la pantalla actual (mensaje de error controlado + reintento si no lo es).
- ✅ Formato correcto de fecha y hora en agendamiento y reprogramación.
- ✅ No se permite agendar ni reprogramar citas en el pasado.
- ✅ Anti-doble-reserva: no se permite una segunda cita del mismo usuario en la misma fecha y hora.
- ✅ Confirmación explícita antes de guardar cualquier cambio (crear, cancelar, completar, reprogramar).
- ✅ Control de permisos por rol (admin vs usuario) antes de acceder a funciones administrativas.

---

## 9. Manejo de errores

El workflow implementa una red de seguridad **nativa de n8n** (sin dependencias de pago ni de la API de n8n):

- Todos los nodos `Cerebro [Módulo]`, `Enviar respuesta [módulo]` y los nodos de escritura en Google Sheets tienen configurado **"On Error: Continue (using error output)"**.
- Cualquier error en esos nodos se redirige a un nodo central: **`Fallback error genérico`**, que:
  1. Envía al usuario un mensaje amigable indicando que algo falló y que fue devuelto al menú principal.
  2. Resetea su sesión en SESIONES (`pantalla_actual = menu_principal`, `paso_actual = 0`, `datos_parciales = {}`).

Esto garantiza que, ante cualquier fallo inesperado, el usuario **nunca queda sin respuesta ni atrapado en un paso roto** — siempre puede continuar usando el bot desde el menú principal.

> **Nota de diseño:** se descartó una arquitectura basada en un workflow de errores separado (Error Trigger + API de n8n) porque la lectura de ejecuciones fallidas vía API está bloqueada en el plan gratuito de n8n Cloud detrás de un muro de pago — incompatible con el requisito del proyecto de no usar nada que requiera tarjeta de crédito.

---

## 10. Registro de usuarios y control de permisos

Al recibir un mensaje, el bot busca al usuario en la hoja USUARIOS por `telegram_user` (chat ID de Telegram):

- **Si no existe:** se crea automáticamente una fila nueva con `rol = "usuario"` y `permitido = "si"`, y el usuario pasa directo al menú principal (acceso abierto para pruebas, según lo definido en el alcance actual del proyecto).
- **Si existe pero `permitido = "no"`:** recibe el mensaje de acceso restringido.
- **Si existe con `permitido = "si"`:** continúa normalmente, con su `rol` (admin / usuario) determinando el acceso al menú de Administrador (opción 8).

---

## 11. Logging y trazabilidad

Cada interacción relevante queda registrada en la hoja LOGS con: marca de tiempo, usuario de Telegram, pantalla en la que ocurrió, opción elegida y resultado de la acción. Esto permite reconstruir la navegación completa de un usuario para efectos de depuración y como evidencia para las pruebas del proyecto.

---

## 12. Limitaciones conocidas

Documentadas de forma transparente para efectos de evaluación:

1. **Configuración** no está implementada — pantalla informativa únicamente.
2. **Recordatorios de hábitos se envían a todos los usuarios permitidos**, no solo al creador del hábito, porque la hoja HABITOS no incluye una columna `creado_por`. Es una limitación estructural aceptada conscientemente dado que, en el alcance actual del proyecto, el bot es usado por un número reducido de personas.
3. El menú de "Recordatorios" (opción 3) es principalmente informativo: explica que el resumen diario y los recordatorios de hábitos ya son automáticos, pero no permite activar/desactivar recordatorios individuales por cita o tarea.

---

## 13. Plan de pruebas

Ver evidencia completa en `evidencias/`. Resumen de la cobertura requerida:

| Tipo de prueba | Cantidad requerida | Carpeta de evidencia |
|---|---|---|
| Navegación por menús | 30 | `evidencias/pruebas_navegacion/` |
| Agendamientos completos | 10 | `evidencias/pruebas_agendamiento/` |
| Errores controlados | 10 | `evidencias/pruebas_errores/` |
| Recordatorios | 10 | `evidencias/pruebas_recordatorios/` |
| Permisos | 10 | `evidencias/pruebas_permisos/` |

Cada prueba debe quedar respaldada por: captura de pantalla de la conversación en Telegram + fila correspondiente en la hoja LOGS.
