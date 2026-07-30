# AgendaBot

Bot conversacional por Telegram para agendar citas, gestionar tareas, hábitos y listas, construido con **Telegram + n8n Community Edition + Google Sheets**, sin depender de plataformas de pago ni APIs que exijan tarjeta de crédito (Art. 2).

## Stack

- **Telegram** — interfaz conversacional con el usuario.
- **n8n Community Edition** — automatización, lógica de negocio y enrutamiento.
- **Google Sheets** — almacenamiento de datos (documento `AgendaBot_DB`).

## Estructura del repositorio

```
├── README.md              # este archivo
├── docs/
│   └── AgendaBot.md        # documentación técnica completa del proyecto
├── workflows/
│   └── (export .json del workflow de n8n)
└── evidencias/
    └── plantilla-pruebas.md # checklist y evidencia de pruebas (Art. 13)
```

## Estado del proyecto

| Módulo | Estado |
|---|---|
| Router principal por pantalla/opción | ✅ Completo y probado |
| Control de permisos y rol admin | ✅ Completo y probado |
| Agenda (CITAS) — crear, consultar, editar, cancelar | ✅ Completo y probado |
| Tareas — crear, consultar, completar | ✅ Completo y probado |
| Resumen diario automático | ✅ Completo y probado |
| Registro de logs | ✅ Completo |
| Hábitos | ✅ Construido (módulo adicional, no exigido por el Art. 11) |
| Listas + ítems | ⚠️ Construido, con un bug conocido pendiente de confirmar (ver `docs/AgendaBot.md` → Problemas conocidos) |
| Menús simples (Ayuda, Recordatorios, Configuración, Informes) | ✅ Mensajes informativos (no exigidos como flujo completo por el Art. 11) |

Para el detalle técnico completo (arquitectura, modelo de datos, decisiones de diseño), ver [`docs/AgendaBot.md`](docs/AgendaBot.md).

## Cómo desplegar

1. Importar el archivo `.json` de `workflows/` en una instancia de n8n Community Edition.
2. Crear el documento de Google Sheets `AgendaBot_DB` con las hojas y columnas descritas en `docs/AgendaBot.md` → Modelo de datos.
3. Configurar las credenciales de Telegram (bot token) y Google Sheets en n8n.
4. Agregar al menos un usuario administrador en la hoja `USUARIOS` (`permitido = si`, `rol = admin`) antes de activar el workflow, o todo intento de acceso será rechazado.
5. Activar/publicar el workflow en n8n.

## Autor

Proyecto individual — Nivel 1 de Automatización con IA.