# Servidor MCP

wacrm incluye un servidor del [Model Context Protocol](https://modelcontextprotocol.io)
para que puedas controlar tu CRM desde asistentes de IA — Claude Desktop,
Claude Code, Cursor y cualquier otro cliente MCP — en lenguaje natural:

> "¿Cuántas conversaciones siguen abiertas hoy?"
> "Muéstrame los últimos cinco mensajes con +1 415 555 0123."
> "Envía la plantilla `order_update` a ese contacto."

Está en [`mcp-server/`](../mcp-server) y se publica en npm como
[`wacrm-mcp`](https://www.npmjs.com/package/wacrm-mcp). Por debajo es un
envoltorio delgado sobre la [API pública](./public-api.md), así que cada
petición se autentica y se acota en tu instancia exactamente igual que
cualquier otra llamada a la API.

## Inicio rápido

1. Crea una clave de API en el panel: **Ajustes → Claves de API**. Otorga
   solo los permisos (scopes) que tu asistente necesite (un asistente de
   solo lectura solo necesita los scopes `*:read`).
2. Agrega el servidor a la configuración de tu cliente MCP:

   ```jsonc
   {
     "mcpServers": {
       "wacrm": {
         "command": "npx",
         "args": ["-y", "wacrm-mcp"],
         "env": {
           "WACRM_BASE_URL": "https://crm.example.com",
           "WACRM_API_KEY": "wacrm_live_xxxxxxxxxxxxxxxxxxxxxxxx"
         }
       }
     }
   }
   ```

Eso es de **solo lectura** — el valor seguro por defecto. Para que el
asistente pueda modificar datos o enviar mensajes, agrega
`"WACRM_ENABLE_WRITES": "true"` (y `"WACRM_ENABLE_BROADCASTS": "true"`
para envíos masivos) en `env`.

## Qué expone

- **Lecturas (siempre activas):** `whoami`, contactos (listar/obtener),
  conversaciones (listar/obtener), mensajes (listar), estado de
  difusiones.
- **Escrituras (opcionales):** enviar un mensaje, crear/actualizar un
  contacto.
- **Difusiones (opcionales):** lanzar una difusión con plantilla — exige
  un `confirm` explícito y está marcada como destructiva.

## Seguridad

Como enviar mensajes de WhatsApp es un efecto secundario real, el
servidor es **de solo lectura hasta que lo habilites**, por encima de los
scopes propios de la clave de API. Dale a un asistente una clave de solo
lectura y una configuración de solo lectura, y físicamente no podrá enviar
nada. Consulta el [README del servidor](../mcp-server/README.md) para ver
la lista completa de herramientas y el modelo de seguridad.
