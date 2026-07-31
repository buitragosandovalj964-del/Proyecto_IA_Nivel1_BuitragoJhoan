# AgendaBot 🤖📅

Bot conversacional de agendamiento y automatización personal, construido con **Telegram + n8n Community Edition + Google Sheets**, sin dependencias de pago ni servicios que requieran tarjeta de crédito.

Proyecto desarrollado por **Jhoan Sebastian Buitrago Sandoval**, estudiante del programa Técnico en Desarrollo de Software — Campuslads (Floridablanca, Colombia).

---

## 📌 ¿Qué hace AgendaBot?

AgendaBot permite, a través de una conversación por Telegram, sin necesidad de aprender comandos:

- 📅 **Agendar, consultar, reprogramar, cancelar y completar citas.**
- ✅ **Gestionar tareas** con prioridad y estado.
- 🔁 **Registrar hábitos** con recordatorios automáticos por hora.
- 📋 **Crear y administrar listas** (compras, pendientes, ideas) con ítems.
- 📊 **Consultar informes** de citas, tareas y hábitos.
- 🌅 **Recibir un resumen diario automático** cada mañana.
- 🔐 **Registro automático de nuevos usuarios**, con control de roles (admin / usuario).

Toda la interacción es 100% guiada por menús numerados — el usuario nunca necesita escribir comandos, solo elegir un número.

---

## 🧱 Stack tecnológico

| Componente | Uso |
|---|---|
| **Telegram Bot API** | Interfaz conversacional con el usuario |
| **n8n (Community Edition)** | Motor de automatización, lógica y orquestación |
| **Google Sheets** | Base de datos (persistencia de citas, tareas, hábitos, listas, usuarios, sesiones y logs) |

Sin n8n Cloud de pago, sin APIs que requieran tarjeta de crédito, sin entrenamiento de modelos ni RAG — tal como exige el reglamento del proyecto.

---

## 📂 Estructura del repositorio

```
Proyecto_IA_Nivel1_BuitragoSandoval/
├── README.md                  ← Este archivo
├── docs/
│   └── AgendaBot.md            ← Documentación técnica completa
├── workflows/
│   └── Agente_Bot_11.json      ← Export del workflow principal de n8n
└── evidencias/
    ├── pruebas_navegacion/     ← Capturas de las 30 pruebas de navegación
    ├── pruebas_agendamiento/   ← Capturas de los 10 agendamientos completos
    ├── pruebas_errores/        ← Capturas de los 10 errores controlados
    ├── pruebas_recordatorios/  ← Capturas de las 10 pruebas de recordatorios
    └── pruebas_permisos/       ← Capturas de las 10 pruebas de permisos
```

---

## 🚀 Cómo levantar el proyecto

1. Importa `workflows/Agente_Bot_11.json` en tu instancia de n8n.
2. Crea un bot en Telegram vía [@BotFather](https://t.me/BotFather) y copia el token.
3. Configura la credencial de Telegram y de Google Sheets OAuth2 dentro de n8n.
4. Duplica la estructura de hojas de `AgendaBot_DB` (ver `docs/AgendaBot.md` → Modelo de datos).
5. Activa el workflow (**Active**, arriba a la derecha del editor).
6. Escríbele "hola" a tu bot en Telegram.

---

## 📖 Documentación técnica completa

Toda la arquitectura, el modelo de datos, los flujos guiados, las validaciones y el manejo de errores están documentados a fondo en:

👉 **[`docs/AgendaBot.md`](./docs/AgendaBot.md)**

---

## ✅ Estado del proyecto

| Módulo | Estado |
|---|---|
| Registro y permisos de usuarios | ✅ Completo |
| Agenda (crear, consultar, reprogramar, cancelar, completar) | ✅ Completo |
| Tareas | ✅ Completo |
| Hábitos + recordatorios automáticos | ✅ Completo |
| Listas | ✅ Completo |
| Informes | ✅ Completo |
| Resumen diario automático | ✅ Completo |
| Manejo de errores (fallback global) | ✅ Completo |
| Configuración | 🟡 En construcción |

---

## 👤 Autor

**Jhoan Sebastian Buitrago Sandoval**
Técnico en Desarrollo de Software · Campuslads
