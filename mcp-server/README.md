# Servidor MCP de wacrm

Un servidor del [Model Context Protocol](https://modelcontextprotocol.io)
para **[wacrm](https://github.com/ArnasDon/wacrm)** — el CRM de WhatsApp
autoalojable. Permite que clientes MCP (Claude Desktop, Claude Code,
Cursor y otros) controlen tu CRM en lenguaje natural:

> "¿Cuántas conversaciones siguen abiertas?"
> "Busca el contacto de +1 415 555 0123 y muestra los últimos mensajes."
> "Redacta y envía una plantilla de actualización de pedido a Jane."

Es un envoltorio delgado sobre la API REST pública
[`/api/v1`](../docs/public-api.md) de wacrm. Toda la autenticación, el
acotamiento y el límite de peticiones los aplica tu instancia de wacrm —
este servidor solo expone la API como herramientas MCP.

## Requisitos previos

1. Una instancia de wacrm en ejecución (tu propio despliegue
   autoalojado).
2. Una clave de API: en el panel ve a **Ajustes → Claves de API → Nueva
   clave de API** y otorga solo los scopes que necesites. La clave se
   muestra una sola vez.

## Instalación y configuración

El servidor lee dos variables de entorno obligatorias y dos protecciones
de escritura opcionales:

| Variable                  | Obligatoria | Propósito                                                          |
| ------------------------- | ----------- | ------------------------------------------------------------------ |
| `WACRM_BASE_URL`          | sí          | URL de tu instancia, p. ej. `https://crm.example.com`              |
| `WACRM_API_KEY`           | sí          | Una clave de API del panel                                         |
| `WACRM_ENABLE_WRITES`     | no          | `true` para exponer escrituras de contactos + envío de mensajes    |
| `WACRM_ENABLE_BROADCASTS` | no          | `true` para exponer difusiones masivas (requiere `WACRM_ENABLE_WRITES`) |

### Claude Desktop / Claude Code / Cursor

Agrégalo a la configuración de tu cliente MCP (p. ej.
`claude_desktop_config.json`, o `.mcp.json` para Claude Code):

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

Esa configuración es de **solo lectura** — el valor seguro por defecto.
Para que el asistente pueda modificar datos o enviar mensajes, agrega las
protecciones de escritura:

```jsonc
"env": {
  "WACRM_BASE_URL": "https://crm.example.com",
  "WACRM_API_KEY": "wacrm_live_xxxxxxxxxxxxxxxxxxxxxxxx",
  "WACRM_ENABLE_WRITES": "true",
  "WACRM_ENABLE_BROADCASTS": "true"
}
```

## Herramientas

Las herramientas de lectura siempre están disponibles. Las de escritura y
difusión aparecen solo cuando se activa su protección.

| Herramienta          | Grupo     | Scope necesario      | Qué hace                                         |
| -------------------- | --------- | -------------------- | ------------------------------------------------ |
| `whoami`             | lectura   | _(cualquier clave válida)_ | Muestra la cuenta + scopes de la clave     |
| `list_contacts`      | lectura   | `contacts:read`      | Lista/busca contactos (paginado)                 |
| `get_contact`        | lectura   | `contacts:read`      | Lee un contacto                                  |
| `list_conversations` | lectura   | `conversations:read` | Lista conversaciones, filtra por estado/contacto |
| `get_conversation`   | lectura   | `conversations:read` | Lee una conversación                             |
| `list_messages`      | lectura   | `messages:read`      | Lista los mensajes de una conversación           |
| `get_broadcast`      | lectura   | `broadcasts:send`    | Consulta el estado de entrega de una difusión    |
| `send_message`       | escritura | `messages:send`      | Envía un mensaje de WhatsApp (texto/plantilla/medio) |
| `create_contact`     | escritura | `contacts:write`     | Crea (busca o crea) un contacto                  |
| `update_contact`     | escritura | `contacts:write`     | Actualiza un contacto / reemplaza sus etiquetas  |
| `send_broadcast`     | difusión  | `broadcasts:send`    | Lanza una difusión con plantilla (requiere `confirm`) |

## Modelo de seguridad

Enviar mensajes de WhatsApp a través de un LLM es un efecto secundario en
el mundo real, así que el servidor aplica tres protecciones en capas:

1. **Solo lectura por defecto.** Las herramientas de escritura y difusión
   ni siquiera se registran — el modelo no puede verlas — salvo que las
   habilites con `WACRM_ENABLE_WRITES` / `WACRM_ENABLE_BROADCASTS`.
2. **Scopes de la clave de API.** Sea lo que sea lo que permitan las
   protecciones, tu instancia de wacrm sigue aplicando los scopes de la
   clave. Una llamada sin el scope adecuado devuelve un error `forbidden`
   limpio. Emite una clave de solo lectura para un asistente de solo
   lectura.
3. **Confirmación explícita de difusión.** `send_broadcast` se niega a
   ejecutarse a menos que se llame con `confirm: true`, y está marcada
   como `destructive` para que los clientes compatibles pidan antes
   confirmación a la persona.

## Desarrollo

```bash
npm install
npm run build      # compila a dist/
npm run typecheck
npm start          # ejecuta el servidor compilado (requiere las variables de entorno)
```

Los logs van a **stderr** — stdout está reservado para el protocolo MCP.

## Licencia

MIT — igual que wacrm.
