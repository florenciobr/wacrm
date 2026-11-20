# API pública (`/api/v1`)

La API pública te permite controlar tu instancia de wacrm desde tus
propios scripts y automatizaciones — enviar mensajes, administrar
contactos, lanzar difusiones — sin pasar por la interfaz del panel.

> **Estado:** estable. La autenticación, los scopes, el límite de
> peticiones, los endpoints de mensajes / contactos / conversaciones /
> difusiones y los [webhooks](#webhooks) de eventos salientes ya están
> disponibles.

## Autenticación

Cada petición se autentica con una **clave de API**, enviada como token
bearer:

```
Authorization: Bearer wacrm_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Las claves están **acotadas a una cuenta**: una clave actúa sobre una
sola cuenta, aquella en la que se creó. No hay acceso entre cuentas.

### Crear una clave

En el panel: **Ajustes → Claves de API → Nueva clave de API**. Solo
**administradores y propietarios** pueden crear claves.

1. Ponle un nombre a la clave (el de la integración que la usará).
2. Otorga los **scopes** que necesita — nada más (ver abajo).
3. Copia la clave. **La clave completa se muestra una sola vez.** wacrm
   guarda únicamente un hash SHA-256, por lo que no puede volver a
   mostrarse. Si la pierdes, revócala y crea una nueva.

### Revocar una clave

**Ajustes → Claves de API → Revocar.** La revocación surte efecto en la
siguiente petición de la clave. Las claves revocadas permanecen en la
lista como registro de auditoría.

## Scopes

Una clave solo puede hacer lo que sus scopes permiten — con independencia
de quién la creó. Otorga el mínimo.

| Scope                | Permite                                       |
| -------------------- | --------------------------------------------- |
| `messages:send`      | Enviar mensajes de WhatsApp                   |
| `messages:read`      | Leer mensajes y estado de entrega             |
| `contacts:read`      | Listar y leer contactos                       |
| `contacts:write`     | Crear y actualizar contactos                  |
| `conversations:read` | Listar y leer conversaciones                  |
| `broadcasts:send`    | Lanzar campañas de difusión                   |
| `webhooks:manage`    | Registrar y administrar webhooks salientes    |

Una clave **sin scopes** igualmente se autentica y puede llamar a
`GET /api/v1/me` — útil para comprobar que una clave funciona.

## Envoltorio de respuesta

Toda respuesta usa una de dos formas:

```jsonc
// éxito
{ "data": { /* ... */ } }

// fallo
{ "error": { "code": "forbidden", "message": "This API key is missing the 'messages:send' scope" } }
```

Ramifica según `error.code` (estable); `error.message` es para personas y
puede reformularse.

| Estado | `code`         | Significado                                          |
| ------ | -------------- | ---------------------------------------------------- |
| 401    | `unauthorized` | Clave ausente / mal formada / desconocida / revocada / expirada |
| 403    | `forbidden`    | Clave válida, pero falta el scope requerido          |
| 429    | `rate_limited` | Se superó el límite de peticiones por clave          |
| 400    | `bad_request`  | Entrada mal formada                                  |
| 404    | `not_found`    | El recurso no existe                                 |
| 500    | `internal`     | Error del servidor                                   |
| 503    | `unavailable`  | No se pudo verificar la clave en este momento — reintenta |

## Límites de peticiones

Las peticiones se limitan **por clave**: **120 peticiones por minuto**.
Ante un `429`, estos encabezados indican cuándo reintentar:

- `Retry-After` — segundos hasta que se reinicie la ventana
- `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

> El limitador está en memoria y es **por proceso**. Un despliegue de una
> sola instancia (el caso habitual de un fork autoalojado) funciona tal
> cual. Si escalas a varias instancias, cambia el limitador por un
> almacén compartido (Redis/Upstash) — consulta la nota al inicio de
> `src/lib/rate-limit.ts`. De lo contrario, el límite no se aplica entre
> instancias.

## Endpoints

### `GET /api/v1/me`

Devuelve la cuenta a la que está vinculada una clave y los scopes que
tiene. Solo requiere una clave válida (sin scope). Úsalo para comprobar
que una clave funciona y para descubrir sus scopes.

```bash
curl https://your-crm.example.com/api/v1/me \
  -H "Authorization: Bearer wacrm_live_xxx"
```

```json
{
  "data": {
    "account": { "id": "…", "name": "Acme Inc" },
    "key": { "id": "…", "scopes": ["messages:send"] }
  }
}
```

### `POST /api/v1/messages`

Envía un mensaje de WhatsApp a un número de teléfono. Scope:
`messages:send`. Se pasa un **número E.164**, no un id interno — el
endpoint busca o crea el contacto + la conversación y luego envía. Un
`name` opcional da nombre al contacto solo cuando esta llamada lo crea;
el nombre de un contacto existente nunca se modifica.

`to` debe comenzar con `+` seguido del código de país (`+14155550123`;
se permiten espacios, guiones y paréntesis entre los dígitos). Un número
sin el `+` se rechaza con `400 bad_request` incluso si por casualidad son
los dígitos correctos: la API no puede distinguir `14155550123` (EE. UU.,
con código de país) de `4155551212` (un número nacional de EE. UU. que
Meta entregaría a +41, Suiza), así que no adivina.

```bash
curl -X POST https://your-crm.example.com/api/v1/messages \
  -H "Authorization: Bearer wacrm_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{ "to": "+14155550123", "type": "text", "text": "Hi 👋" }'
```

`type` es `text` (por defecto), `template` o un tipo de medio (`image` /
`video` / `document` / `audio`). Los medios requieren `media_url` (y
opcionalmente `filename`); `text` hace de pie de foto. `template` requiere
un objeto `template`:

```jsonc
{
  "to": "+14155550123",
  "type": "template",
  "template": {
    "name": "order_update",
    "language": "en_US",
    "params": ["A123"]        // variables posicionales del cuerpo, o un objeto estructurado
  },
  "reply_to_message_id": "<uuid>"   // opcional; debe estar en la misma conversación
}
```

Respuesta (201):

```json
{
  "data": {
    "message_id": "…",
    "whatsapp_message_id": "wamid.…",
    "conversation_id": "…",
    "contact_id": "…",
    "contact_created": true
  }
}
```

Códigos de error de dominio además de los de la tabla anterior:
`whatsapp_not_configured` (400), `meta_error` (502 — la petición llegó a
Meta y esta rechazó el envío), `template_malformed` (500).

### `GET /api/v1/contacts`

Lista contactos, del más reciente al más antiguo. Scope: `contacts:read`.
Paginado (ver [Paginación](#paginación)). Filtros opcionales: `?search=`
(coincide con nombre o teléfono) y `?tag=<tagId>`.

```json
{
  "data": [
    {
      "id": "…", "phone": "+14155550123", "name": "Jane Doe",
      "email": null, "company": "Acme", "avatar_url": null,
      "tags": [{ "id": "…", "name": "vip", "color": "#3b82f6" }],
      "created_at": "…", "updated_at": "…"
    }
  ],
  "meta": { "next_cursor": "…" }
}
```

### `POST /api/v1/contacts`

Crea un contacto. Scope: `contacts:write`. `phone` (E.164, se requiere el
`+` inicial y el código de país — misma regla que `POST /api/v1/messages`)
es obligatorio; `name`, `email`, `company` y `tags` (un arreglo de
nombres de etiqueta, que se crean si no existen) son opcionales.
**Buscar o crear por teléfono:** si ya existe, devuelve `200` con el
contacto existente; un contacto nuevo devuelve `201`. En un contacto
existente, `name`, `email` y `company` no se modifican y las `tags` se
**agregan** a las que ya tiene (no se quita ninguna); usa `PATCH` para
reemplazar sus etiquetas. El cuerpo de la respuesta es el contacto
serializado (misma forma que las filas del listado anterior).

### `GET` / `PATCH /api/v1/contacts/{id}`

Lee o actualiza un contacto. Scopes: `contacts:read` / `contacts:write`.
`PATCH` actualiza solo los campos que envíes (`name`, `email`,
`company`); pasa `tags` (un arreglo de nombres de etiqueta) para
reemplazar las etiquetas del contacto. Un contacto de otra cuenta
devuelve `404`.

### `GET /api/v1/conversations`

Lista conversaciones, de la más reciente a la más antigua. Scope:
`conversations:read`. Paginado. Filtros opcionales: `?status=` (`open` /
`pending` / `closed`) y `?contact_id=`. Cada conversación incluye su
contacto + etiquetas.

### `GET /api/v1/conversations/{id}`

Lee una conversación. Scope: `conversations:read`. `404` si pertenece a
otra cuenta.

### `GET /api/v1/conversations/{id}/messages`

Lista los mensajes de una conversación, del más reciente al más antiguo.
Scope: `messages:read`. Paginado. Cada mensaje incluye su `direction`
(`inbound` / `outbound`), `status` (estado de entrega),
`whatsapp_message_id` y `content_*`. Primero se verifica que la
conversación pertenezca a tu cuenta (`404` en caso contrario).

### `POST /api/v1/broadcasts`

Lanza una difusión con plantilla a una lista de destinatarios. Scope:
`broadcasts:send`. La difusión y sus filas de destinatarios se guardan de
inmediato y los envíos se distribuyen en segundo plano, así que la
llamada responde rápido — consulta `GET /api/v1/broadcasts/{id}` para ver
el progreso.

```bash
curl -X POST https://your-crm.example.com/api/v1/broadcasts \
  -H "Authorization: Bearer wacrm_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "July promo",
        "template_name": "promo_july",
        "template_language": "en_US",
        "recipients": [
          { "to": "+14155550123", "params": ["Jane"] },
          { "to": "+14155550124" }
        ]
      }'
```

Los destinatarios están limitados a **1000 por petición** — divide los
envíos más grandes. Los números de teléfono no válidos — incluidos los
que no tengan `+` inicial y código de país — se descartan y se cuentan
como `rejected`; si ninguno es válido, la petición falla con
`400 bad_request`. Los destinatarios que resuelvan a un contacto ya
listado antes en la misma petición reciben el mensaje una sola vez y se
cuentan como `duplicates`. Respuesta (202):

```json
{
  "data": {
    "broadcast_id": "…",
    "status": "sending",
    "total_recipients": 2,
    "accepted": 2,
    "rejected": 0,
    "duplicates": 0
  }
}
```

### `GET /api/v1/broadcasts/{id}`

Estado y conteos de la difusión. Scope: `broadcasts:send`. `status` pasa
de `sending` a `sent`; `delivered_count` / `read_count` siguen subiendo a
medida que llegan los webhooks de entrega de Meta. `404` para la difusión
de otra cuenta.

## Paginación

Todos los endpoints de listado paginan igual. Solicita un tamaño de
página con `?limit=` (por defecto 50, máximo 100) y obtén la página
siguiente con el `meta.next_cursor` opaco de la respuesta anterior:

```
GET /api/v1/contacts?limit=50
→ { "data": [ … ], "meta": { "next_cursor": "eyJ…" } }

GET /api/v1/contacts?limit=50&cursor=eyJ…
→ { "data": [ … ], "meta": { "next_cursor": null } }   // última página
```

Los cursores se basan en keyset (estables ante inserciones concurrentes).
Devuelve el cursor tal cual — no lo analices. `next_cursor: null` indica
la última página. Un cursor mal formado o manipulado se ignora y se
devuelve la primera página.

## Webhooks

En lugar de consultar periódicamente, registra un endpoint y wacrm hará
POST a él cuando ocurran cosas en tu cuenta. **Requiere migración:**
aplica `supabase/migrations/028_webhook_endpoints.sql`.

### Eventos

| Evento                   | Se dispara cuando                                 |
| ------------------------ | ------------------------------------------------- |
| `message.received`       | Llega un mensaje entrante de un contacto          |
| `message.status_updated` | Un mensaje que enviaste cambió de estado de entrega |
| `conversation.created`   | Se abre una conversación nueva para un contacto (por un mensaje entrante o `POST /api/v1/messages`) |

### Administrar endpoints

Todos bajo el scope `webhooks:manage`.

- `POST /api/v1/webhooks` — registra `{ "url": "https://…", "events": ["message.received"] }`. `url` debe ser `https://` y no debe incrustar usuario ni contraseña (`https://user:pass@…` se rechaza con `400`; verifica las entregas con el secreto de firma). **La respuesta incluye `secret` una sola vez** — guárdalo para verificar firmas; wacrm conserva solo una copia cifrada.
- `GET /api/v1/webhooks` — lista tus endpoints (nunca devuelve el secreto).
- `GET /api/v1/webhooks/{id}` — lee uno.
- `PATCH /api/v1/webhooks/{id}` — actualiza `url`, `events` o `is_active` (al reactivarlo se reinicia el contador de fallos).
- `DELETE /api/v1/webhooks/{id}` — elimina uno.

```bash
curl -X POST https://your-crm.example.com/api/v1/webhooks \
  -H "Authorization: Bearer wacrm_live_xxx" \
  -H "Content-Type: application/json" \
  -d '{ "url": "https://example.com/hooks/wacrm", "events": ["message.received"] }'
# → 201 { "data": { "id": "…", "url": "…", "events": [...], "secret": "whsec_…" } }
```

### Contenido de la entrega

Cada entrega es un POST con este envoltorio; `id` es un uuid único por
entrega que puedes usar para deduplicar, y `data` varía según `event`:

```json
{
  "id": "8f3c…",
  "event": "message.received",
  "occurred_at": "2026-07-01T12:00:00.000Z",
  "account_id": "…",
  "data": { /* por evento, ver abajo */ }
}
```

`data` por evento:

```jsonc
// message.received
{ "conversation_id": "…", "contact_id": "…", "whatsapp_message_id": "wamid.…", "content_type": "text", "text": "Hi 👋" }
// conversation.created
{ "conversation_id": "…", "contact_id": "…" }
// message.status_updated
{ "whatsapp_message_id": "wamid.…", "conversation_id": "…", "status": "delivered" }
```

Encabezados: `X-Wacrm-Event`, `X-Wacrm-Webhook-Id` y `X-Wacrm-Signature`.

### Verificar la firma

`X-Wacrm-Signature: t=<unix_seconds>,v1=<hex>` donde `v1 =
HMAC-SHA256(secret, "${t}.${rawBody}")`. Recalcúlala sobre el **cuerpo
crudo de la petición** y compara en tiempo constante; rechaza si `t`
tiene más de unos pocos minutos de antigüedad (protección contra
repetición).

```js
const [, t, v1] = header.match(/t=(\d+),v1=([0-9a-f]+)/);
const expected = crypto.createHmac('sha256', secret)
  .update(`${t}.${rawBody}`).digest('hex');
const ok = crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(v1));
```

### Semántica de entrega

La entrega es de **mejor esfuerzo**: un solo intento por evento con un
tiempo de espera corto, y **no se siguen las redirecciones**.
`message.status_updated` cubre los mensajes que wacrm almacena (bandeja +
envíos por API), no los envíos solo de difusión, y — como los
proveedores reenvían y reordenan los callbacks de estado — el mismo
estado puede llegar más de una vez o desordenado; **deduplica por `id` y
no asumas orden**. Cada fallo consecutivo incrementa `failure_count`;
tras suficientes fallos consecutivos el endpoint se desactiva
automáticamente (`is_active: false`) — reactívalo con `PATCH` (que
reinicia el contador). El reintento duradero con retroceso (una cola de
entregas) es una mejora futura; hoy, considera posibles las entregas
perdidas y reconcilia con los endpoints de lectura cuando importe.

**Restricciones de destino (SSRF).** La `url` debe ser `https://` y
resolver a una dirección pública — las peticiones a `localhost`, rangos
privados/RFC1918, link-local (incluido el metadato de nube
`169.254.169.254`) y destinos internos similares se rechazan en el
momento de la entrega.

## Hoja de ruta

La API pública ya cubre mensajería, contactos, conversaciones,
difusiones y webhooks salientes — el alcance completo de
[#245](https://github.com/ArnasDon/wacrm/issues/245). Ideas futuras
(negocios/embudos, plantillas, flujos, una cola de entregas para
webhooks) aún no están programadas.
