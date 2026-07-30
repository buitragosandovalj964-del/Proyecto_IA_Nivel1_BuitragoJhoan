# AgendaBot — Documentación técnica

## 1. Introducción (Art. 1)

AgendaBot Services requiere un bot automatizado que permita agendar, planificar y automatizar tareas básicas, sin depender de plataformas de pago ni servicios que exijan tarjeta de crédito. Este documento describe la implementación real del proyecto, su arquitectura, modelo de datos, decisiones de diseño y estado de cumplimiento frente a los requisitos originales.

## 2. Stack tecnológico (Art. 2)

- Telegram (interfaz conversacional).
- n8n Community Edition (automatización y lógica).
- Google Sheets (almacenamiento de datos).

No se usó n8n Cloud de pago, ni APIs que requieran tarjeta de crédito, ni entrenamiento de modelos, embeddings o RAG.

## 3. Enfoque conversacional (Art. 3)

El bot sigue estos principios en todos sus flujos:

- El usuario siempre elige qué hacer escribiendo un número.
- El bot siempre explica qué hace y qué opciones hay.
- El bot siempre sugiere una opción recomendada.
- El bot nunca asume intención.
- El bot siempre ofrece salida (`9` para volver o cancelar, disponible en cualquier paso de cualquier wizard).

## 4. Modelo de datos (Art. 4)

Documento de Google Sheets `AgendaBot_DB`, con las siguientes hojas:

| Hoja | Columnas |
|---|---|
| CITAS | `id_cita, fecha, hora, nombre, motivo, canal, estado, creado_por, timestamp_creacion` |
| TAREAS | `id_tarea, titulo, prioridad, estado, fecha_objetivo, creado_por` |
| HABITOS | `id_habito, nombre, frecuencia, hora_recordatorio, estado` |
| LISTAS | `id_lista, nombre_lista, tipo, creado_por` |
| ITEMS_LISTA | `id_item, id_lista, item, estado` |
| USUARIOS | `telegram_user, nombre, rol, permitido` |
| LOGS | `timestamp, telegram_user, pantalla, opcion_elegida, resultado` |
| SESSIONS | `telegram_user, pantalla_actual, paso_actual, datos_parciales, timestamp_ultima_interaccion` |

No se agregaron columnas fuera de esta estructura en ningún módulo.

## 5. Arquitectura del workflow

### 5.1 Router principal
Un nodo Switch enruta cada mensaje entrante según la `pantalla_actual` guardada en `SESSIONS` para ese `telegram_user`. Un segundo Switch (`Switch1`) enruta las opciones numéricas del menú principal hacia cada módulo.

### 5.2 Control de acceso
Antes de procesar cualquier mensaje:
1. Se busca al usuario en `SESSIONS` (sesión existente o nueva) y en `USUARIOS` (autorización).
2. Un nodo Code (`Normalizar sesion`) unifica ambos resultados con comparación robusta de tipos (texto/número) y de valores de verdad (`si`, `true`, `1`, `x`, `yes`, `verdadero` se consideran todos como autorización concedida).
3. Un If (`Tiene permiso?`) bloquea el flujo con un mensaje de acceso restringido si el usuario no está autorizado, registrando el intento en `LOGS`.
4. Los usuarios con `rol = admin` acceden además a un panel de administrador (opción 8 del menú principal).

### 5.3 Patrón de wizard multi-paso
Todos los flujos de captura de datos (nueva cita, nueva tarea, nuevo hábito, nueva lista/ítem) siguen el mismo patrón:

- `SESSIONS.pantalla_actual` identifica en qué wizard está el usuario.
- `SESSIONS.paso_actual` identifica el paso dentro del wizard.
- `SESSIONS.datos_parciales` acumula, en formato JSON, los datos capturados hasta el momento.
- Un nodo Code por wizard (la "máquina de pasos") valida la respuesta del usuario para el paso actual, y decide si: pide el siguiente dato, informa un error de validación y repite el mismo paso, o llega al paso de confirmación.
- El usuario puede escribir `9` en cualquier paso para cancelar sin guardar nada.
- Al confirmar, se genera un ID único (`CITA-`, `TAREA-`, etc. + timestamp) y se escribe la fila correspondiente en su hoja de Google Sheets.
- Cada transición relevante queda registrada en `LOGS`.

## 6. Mensajería (Art. 5, 6, 7, 8, 9)

Todos los mensajes del bot siguen la estructura: saludo cercano, explicación breve, opciones numeradas, sugerencia, indicación de cómo continuar o salir — con uso moderado de emojis para mantener un tono cercano sin perder profesionalismo.

El menú principal, el mensaje de bienvenida y el mensaje de "opción inválida" siguen el texto literal definido en los Artículos 6, 7 y 8 del documento de requisitos original.

## 7. Módulos implementados

### 7.1 Agenda / CITAS (Art. 9, 10) — completo
Wizard de 6 pasos (fecha, hora, nombre, motivo, canal, confirmación), con las siguientes validaciones (Art. 12):
- Formato de fecha y hora correctos.
- No permite agendar en el pasado.
- Evita doble reserva (misma fecha y hora ya existente antes de guardar).
- Confirmación explícita antes de guardar.

Opciones: agendar, consultar, reprogramar, cancelar, marcar como completada.

### 7.2 Tareas — completo
Wizard de 4 pasos (título, prioridad, fecha objetivo con validación de no pasada, confirmación). Opciones: crear, consultar pendientes (filtradas por usuario y estado), marcar como completada (por ID, validando pertenencia al usuario).

Este módulo cumple explícitamente el requisito del Art. 11: "flujo de tareas con estados".

### 7.3 Resumen diario automático (Art. 11) — completo
Un Schedule Trigger diario (7:00 a.m.) recorre `USUARIOS` permitidos, y para cada uno construye y envía por Telegram un resumen de sus citas próximas y tareas pendientes, registrando el envío en `LOGS`.

### 7.4 Hábitos — construido (módulo adicional, no exigido por el Art. 11)
Wizard de creación (nombre, frecuencia, hora de recordatorio con validación de formato), consulta, marcar cumplido y eliminar (soft-delete vía `estado = eliminado`).

> **Nota de diseño:** el modelo de datos oficial de `HABITOS` no incluye una columna para registrar el historial de cumplimientos. Para no apartarse del Art. 4 (no inventar columnas), cada evento de "cumplido hoy" se registra como una fila en `LOGS` en lugar de una columna nueva. Si se requiere un historial de cumplimiento consultable, sería necesario ampliar el modelo de datos de forma explícita.

### 7.5 Listas + ítems — construido, con problema conocido
Wizard de creación de lista, agregar ítems, ver listas con sus ítems, marcar ítem completado, eliminar lista. Ver sección 9 (Problemas conocidos).

### 7.6 Menús informativos — Ayuda, Recordatorios, Configuración, Informes
Ayuda es funcional (explica el bot y sugiere por dónde empezar). Recordatorios y Configuración son mensajes informativos que indican que la función está en construcción, ya que no son exigidos como flujo completo por el Art. 11. Informes sí quedó implementado como conteos reales sobre las hojas de CITAS, TAREAS y HABITOS.

## 8. Automatizaciones obligatorias (Art. 11) — checklist de cumplimiento

- [x] Router principal por pantalla y opción numérica.
- [x] Flujo guiado de agendamiento.
- [x] Flujo de tareas con estados.
- [x] Resumen diario por Telegram.
- [x] Registro automático de logs.

## 9. Problemas conocidos

- **Módulo Listas — hoja `ITEMS_LISTA`:** durante el desarrollo se detectó un desajuste entre el nombre de la pestaña en Google Sheets (`ITEM_LISTAS`) y el nombre esperado por los nodos (`ITEMS_LISTA`), causando el error `Sheet with name ITEMS_LISTA not found` y dejando el bot sin responder dentro de este módulo. Se corrigió el nombre de la pestaña; **queda pendiente confirmar en producción** que la referencia de los 3 nodos afectados (`Get ITEMS_LISTA`, `Guardar item en ITEMS_LISTA`, `Actualizar item en ITEMS_LISTA`) se haya refrescado correctamente tras el renombrado.

## 10. Pruebas (Art. 13)

Ver `evidencias/plantilla-pruebas.md` para la plantilla y el registro de evidencia de las pruebas requeridas: 30 de navegación por menús, 10 agendamientos completos, 10 errores controlados, 10 pruebas de recordatorios, 10 pruebas de permisos.

## 11. Entregables (Art. 15)

Repositorio privado `Proyecto_IA1_ApellidoNombre`, compartido con el Trainer, con la estructura descrita en el `README.md` de la raíz del repositorio.
