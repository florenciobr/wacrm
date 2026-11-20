# Varias cuentas de WhatsApp Business en un mismo despliegue de wacrm

El issue #500 preguntó cómo un solo despliegue puede atender cuentas cuyos
números pertenecen a distintas cuentas de WhatsApp Business (WABA), dado
que `META_APP_SECRET` y `META_APP_ID` son variables de entorno de todo el
despliegue. Respuesta corta: las WABA ya son por cuenta; solo la *App* de
Meta es de todo el despliegue, y desde #500 el webhook acepta varios
secretos de app. Esta página recorre las tres configuraciones.

## Qué es por cuenta y qué es por despliegue

| Valor | Dónde vive | Alcance |
| --- | --- | --- |
| Phone Number ID, WABA ID, token de acceso, token de verificación, PIN de dos pasos | `whatsapp_config` (una fila por cuenta de wacrm, token cifrado) | por cuenta |
| URL de callback del webhook | tu App de Meta → WhatsApp → Configuration | por App de Meta |
| `META_APP_SECRET` | entorno del servidor | por despliegue — **admite varios** |
| `META_APP_ID` | entorno del servidor | por despliegue — valor único |

Meta firma cada entrega de webhook con el secreto de la App a la que está
suscrita la WABA. wacrm verifica esa firma contra `META_APP_SECRET` antes
de mirar el cuerpo, y luego enruta el contenido a la cuenta cuya fila
contiene el `phone_number_id` de la entrega.

## Configuración A — muchas cuentas, una App de Meta, muchas WABA (funciona de fábrica)

Es el caso normal de una agencia o de una empresa con varias marcas. No
hay nada que configurar más allá de una instalación de un solo inquilino.

1. Crea una App de Meta y agrega el producto WhatsApp. En
   **WhatsApp → Configuration** define la URL de callback como
   `https://<tu host>/api/whatsapp/webhook` y suscríbete al campo
   `messages`. Define `META_APP_SECRET` con el secreto de esta app.
2. Para cada WABA, asegúrate de que el portafolio de negocio que la posee
   tenga acceso a la app (Business Settings → Accounts → WhatsApp
   accounts → Assigned assets, o agrega la WABA mediante el producto
   WhatsApp de la app).
3. Cada cuenta de wacrm abre Ajustes → Conexión de WhatsApp e ingresa su
   propio Phone Number ID, WABA ID, un token de acceso de System User que
   pueda administrar *esa* WABA, un token de verificación y (para números
   de producción) el PIN de dos pasos.

Al guardar, wacrm verifica el número con el token, lo registra
(`POST /{phone_number_id}/register`, requiere PIN) y suscribe la WABA a la
app (`POST /{waba_id}/subscribed_apps`). A partir de ahí Meta entrega los
eventos de todas las WABA a la única URL de callback y wacrm los reparte
por `phone_number_id`.

Restricciones:

* Un número de teléfono solo puede conectarse a una cuenta de wacrm — la
  ruta rechaza un `phone_number_id` ya reclamado por otra cuenta.
* El **token de verificación** escrito en la configuración del webhook de
  la App de Meta debe ser igual al token de verificación guardado por al
  menos una cuenta de wacrm; el handshake (`GET /api/whatsapp/webhook`)
  acepta el token de cualquier cuenta. Usar la misma cadena en todas las
  cuentas lo mantiene simple.

## Configuración B — WABA bajo distintas Apps de Meta

A veces las WABA no pueden compartir una app: dos clientes tienen cada uno
su propia App de Meta y portafolio, o no se puede dar a una app existente
acceso a la WABA de otro negocio. Cada app firma los webhooks con su
propio secreto, así que wacrm necesita todos.

1. En **cada** App de Meta, define la misma URL de callback
   (`https://<tu host>/api/whatsapp/webhook`) y suscríbete a `messages`.
   Usa un token de verificación que al menos una cuenta de wacrm haya
   guardado (ver la nota anterior).
2. Define `META_APP_SECRET` con la lista, separada por comas, de los
   secretos de todas esas apps:

   ```
   META_APP_SECRET=secret-of-app-one,secret-of-app-two
   ```

   Los espacios alrededor de las comas se ignoran y los huecos vacíos se
   descartan. Una entrega se acepta cuando su `X-Hub-Signature-256`
   coincide con **cualquiera** de los secretos listados; cada comparación
   es de tiempo constante. Un valor vacío o con solo comas sigue fallando
   de forma cerrada — se rechaza toda petición.
3. Cada cuenta de wacrm conecta su número exactamente como en la
   configuración A, usando un token generado dentro del portafolio de
   negocio que posee su app. Al guardar, se suscribe la WABA a *esa* app
   (aquella a la que pertenece el token).

Rotar un secreto: agrega el nuevo a la lista, vuelve a desplegar, luego
rota en Meta y después quita el antiguo. Las entregas siguen
verificándose durante todo el proceso.

## Configuración C — lo que no se admite: un App ID de Meta por cuenta

`META_APP_ID` se usa para una sola cosa: la Resumable Upload de Meta
cuando una plantilla de mensaje tiene un **encabezado de imagen** (la
muestra del encabezado debe ser un identificador de carga con alcance de
app, no una URL). Es un valor único.

Con la configuración B, la carga de plantillas con encabezado de imagen de
cada cuenta pasa entonces por la única app indicada en `META_APP_ID`,
autenticada con el token de esa cuenta. Si el token de la cuenta se generó
bajo el portafolio de otra app, Meta rechaza la carga y el envío de la
plantilla devuelve ese error. Las plantillas de solo texto, el envío, la
recepción y todo lo demás no se ven afectados.

Soluciones temporales hasta que exista un app id por cuenta:

* Poner las WABA que necesiten plantillas con encabezado de imagen bajo la
  app indicada en `META_APP_ID`; o
* Crear las plantillas con encabezado de imagen directamente en WhatsApp
  Manager — una vez aprobadas, wacrm las lista y las envía como cualquier
  otra plantilla.

Una columna `app_id` por cuenta en `whatsapp_config` eliminaría esta
limitación; no está planeada a corto plazo porque el único consumidor es
la carga del encabezado de plantillas.

## Verificar un despliegue con varias WABA

* Ajustes → Conexión de WhatsApp → **Verify with Meta** ejecuta, por
  cuenta, el diagnóstico por verificación (`phone_metadata_ok`,
  `waba_subscribed_to_app`, `locally_marked_registered`).
  `waba_subscribed_to_app` en falso significa que al guardar nunca se
  logró suscribir la WABA a su app — vuelve a ingresar el token y guarda
  de nuevo.
* Una entrega de webhook rechazada con 401 significa que ningún secreto
  configurado produjo su firma — la WABA está suscrita a una app cuyo
  secreto no está en `META_APP_SECRET`.
