# Solución de problemas de la conexión con WhatsApp

Ajustes → Conexión de WhatsApp habla con la Graph API de Meta en un
orden fijo cuando haces clic en **Guardar configuración**. Cuando una de
esas llamadas falla, wacrm ahora te dice *cuál* llamada falló, *qué
campo* revisar, y te da el código de error de Meta y el trace id para
citar al soporte de Meta (issue #505). Esta página lista las causas
comunes y el texto exacto que wacrm muestra para cada una.

> Nota: los mensajes citados abajo están en inglés tal como los emite la
> aplicación; con `NEXT_PUBLIC_APP_LOCALE=es` pueden aparecer traducidos.

## Qué hace Guardar configuración

| Paso (se muestra en los detalles del error) | Llamada a Meta | Demuestra |
| --- | --- | --- |
| *(antes de cualquier llamada a Meta)* | — | Que Phone Number ID y WABA ID son cadenas de dígitos |
| `verify_number` | `GET /{phone_number_id}` | que el token puede leer el número |
| `waba_phone_numbers` | `GET /{waba_id}/phone_numbers` | que el número realmente pertenece a esa WABA |
| `register` | `POST /{phone_number_id}/register` (solo cuando se ingresa un PIN) | que el número está registrado en la Cloud API para webhooks entrantes |
| `subscribe_waba` | `POST /{waba_id}/subscribed_apps` | que la WABA está suscrita a tu App de Meta, de modo que se entreguen los webhooks |

Los primeros cuatro fallos detienen el guardado; no se escribe nada hasta
que se corrija la causa. Un fallo de `register` sí guarda las
credenciales (para que puedas reintentar solo con el PIN) y muestra el
motivo en el aviso *Registration status*.

**Test API Connection** vuelve a ejecutar `verify_number` con el token
guardado y, cuando hay un WABA ID registrado, lee
`GET /{waba_id}/subscribed_apps` e informa si la WABA está suscrita a una
app. Credenciales válidas más una WABA sin suscribir es exactamente el
estado de "conectado pero no llegan mensajes".

## Cómo leer los detalles del error

Bajo el mensaje accionable, un texto pequeño y atenuado muestra:

```
Step: verify_number · Meta error code: 190/463 · Trace ID: AbCdEf...
Meta said: Error validating access token: Session has expired ...
```

* **Step** — la fila de la tabla anterior.
* **Meta error code** — `code/subcode` del envoltorio de Meta. La
  [referencia de códigos de error de la Cloud API](https://developers.facebook.com/docs/whatsapp/cloud-api/support/error-codes)
  documenta cada valor.
* **Trace ID** — el `fbtrace_id` de Meta. El soporte de Meta puede
  localizar la petición con él; cítalo junto con el código.

El estado HTTP también indica quién debe actuar: **400** significa que la
corrección está en el formulario de ajustes (token, ids, PIN); **502**
significa que Meta tiene que cambiar algo (límite de peticiones,
restricción, caída).

## Causas comunes y qué muestra wacrm

### El Phone Number ID no es un id

*Causa:* se pegó el número de teléfono (`+1 555 ...`) o un nombre visible
en lugar del id numérico.

> Phone Number ID must contain only digits — it is the numeric id shown
> under Meta → WhatsApp → API Setup, not the phone number itself.

(El Phone Number ID debe contener solo dígitos — es el id numérico que
aparece en Meta → WhatsApp → API Setup, no el número de teléfono.)

La misma verificación se aplica al WABA ID:

> WhatsApp Business Account ID must contain only digits — copy it from
> Meta → WhatsApp → API Setup.

(El WhatsApp Business Account ID debe contener solo dígitos — cópialo de
Meta → WhatsApp → API Setup.)

### Token de acceso expirado o no válido (código 190)

*Causa:* se usó el token temporal de 24 horas de la página API Setup, o
se revocó / restableció un token de System User.

> The access token has expired. Temporary tokens from the API Setup
> page expire after 24 hours. Generate a permanent token in Meta
> Business Settings → System Users → Generate token, with the
> whatsapp_business_management and whatsapp_business_messaging
> permissions, and paste it into Permanent Access Token.

(El token de acceso expiró. Los tokens temporales de la página API Setup
caducan a las 24 horas. Genera un token permanente en Meta Business
Settings → System Users → Generate token, con los permisos
whatsapp_business_management y whatsapp_business_messaging, y pégalo en
Permanent Access Token.)

(La primera frase varía según el subcódigo: "has been invalidated" para
460/467, "Meta rejected the access token as invalid" en los demás casos.)

### El token carece de permisos (código 10, 200–299)

*Causa:* el token de System User se generó sin
`whatsapp_business_management` / `whatsapp_business_messaging`, o el
System User nunca se asignó a la WABA.

> The access token is not allowed to perform this action (subscribing
> the WhatsApp Business Account to the app). Its System User needs the
> whatsapp_business_management and whatsapp_business_messaging
> permissions AND must be assigned to this WhatsApp Business Account
> (Business Settings → System Users → Add assets → WhatsApp accounts).
> Then generate a new token.

(El token de acceso no tiene permiso para esta acción (suscribir la
cuenta de WhatsApp Business a la app). Su System User necesita los
permisos whatsapp_business_management y whatsapp_business_messaging Y
debe estar asignado a esta cuenta de WhatsApp Business (Business Settings
→ System Users → Add assets → WhatsApp accounts). Luego genera un token
nuevo.)

El código 131005 (acceso denegado) muestra un mensaje similar que nombra
el id rechazado.

### Phone Number ID o WABA ID incorrecto (código 100 "Unsupported get request", código 33)

*Causa:* el id está mal escrito, o pertenece a un portafolio de negocio
distinto de aquel en el que se generó el token.

> Meta cannot find Phone Number ID 1234567890, or the business that
> owns the access token does not own it. Copy the Phone Number ID
> exactly from Meta → WhatsApp → API Setup and check the token was
> generated inside the same Business portfolio.

(Meta no encuentra el Phone Number ID 1234567890, o el negocio dueño del
token de acceso no es dueño de él. Copia el Phone Number ID exactamente
de Meta → WhatsApp → API Setup y comprueba que el token se generó dentro
del mismo portafolio de negocio.)

Cuando el paso que falla es de alcance WABA, el mensaje nombra el
*WhatsApp Business Account ID* en su lugar.

### El número pertenece a otra WABA

*Causa:* ambos ids son válidos, pero la WABA que escribiste no es dueña
del número. Antes de #505 esto se guardaba sin problema y suscribía la
WABA equivocada — el webhook simplemente nunca se disparaba.

> Phone Number ID 111 does not belong to WhatsApp Business Account 999.
> Meta lists these numbers under it: +1 555-0100 (222), +1 555-0200
> (333). Check both values in Meta → WhatsApp → API Setup: the WABA ID
> shown there must be the one that lists this phone number.

(El Phone Number ID 111 no pertenece a la cuenta de WhatsApp Business
999. Meta lista bajo ella estos números: +1 555-0100 (222), +1 555-0200
(333). Revisa ambos valores en Meta → WhatsApp → API Setup: el WABA ID
que se muestra ahí debe ser el que lista este número de teléfono.)

### Número no registrado en la Cloud API (código 133010)

> This phone number is not registered with the WhatsApp Cloud API yet.
> Enter the two-step verification PIN below and save again so wacrm can
> register it (POST /register).

(Este número aún no está registrado en la WhatsApp Cloud API. Ingresa
abajo el PIN de verificación en dos pasos y guarda de nuevo para que
wacrm pueda registrarlo (POST /register).)

### PIN de dos pasos incorrecto (códigos 133005, 136025)

> The two-step verification PIN is wrong. Use the 6-digit PIN set in
> WhatsApp Manager → Phone numbers → Two-step verification (or reset it
> there), then save again.

(El PIN de verificación en dos pasos es incorrecto. Usa el PIN de 6
dígitos definido en WhatsApp Manager → Phone numbers → Two-step
verification (o restablécelo ahí) y guarda de nuevo.)

Demasiados intentos fallidos (133008 / 133009) bloquean los intentos por
un tiempo:

> Meta has temporarily locked PIN attempts for this number after too
> many wrong guesses. Wait a while before saving again with the correct
> PIN.

(Meta bloqueó temporalmente los intentos de PIN de este número tras
demasiados errores. Espera un rato antes de guardar de nuevo con el PIN
correcto.)

### Cuenta restringida por Meta (código 131031, 368)

Nada en wacrm lo soluciona — es una política o un estado de verificación
de Meta.

> Meta has restricted or locked this WhatsApp Business Account, so
> nothing in wacrm can connect it. Open Meta Business Manager → Account
> quality (or WhatsApp Manager → Overview) to see the restriction and
> appeal it.

(Meta restringió o bloqueó esta cuenta de WhatsApp Business, por lo que
nada en wacrm puede conectarla. Abre Meta Business Manager → Account
quality (o WhatsApp Manager → Overview) para ver la restricción y
apelarla.)

### Límite de peticiones (códigos 4, 17, 32, 613, 80007, 130429, …)

> Meta is rate-limiting this app or WhatsApp Business Account right
> now. Nothing needs changing — wait a few minutes and try again.

(Meta está limitando las peticiones de esta app o cuenta de WhatsApp
Business en este momento. No hay que cambiar nada — espera unos minutos y
vuelve a intentarlo.)

### Fallo temporal del lado de Meta (códigos 1, 2, 131000, 133004, 133016)

> Meta returned a temporary error while reading the phone number (code
> 131000). Retry in a minute; if it keeps happening, check
> metastatus.com and quote the trace id to Meta support.

(Meta devolvió un error temporal al leer el número (código 131000).
Reintenta en un minuto; si sigue ocurriendo, revisa metastatus.com y cita
el trace id al soporte de Meta.)

### Cualquier otro caso

Los códigos desconocidos conservan las palabras de Meta más el trace id:

> Meta returned an error while registering the phone number (code
> 12345/6): <mensaje de Meta>. Trace id AbCdEf.

Si el servidor no puede alcanzar `graph.facebook.com` en absoluto:

> Could not reach the Meta Graph API while reading the phone number:
> fetch failed. Check that this server has outbound internet access to
> graph.facebook.com and try again.

(No se pudo alcanzar la Graph API de Meta al leer el número: fetch
failed. Comprueba que este servidor tenga acceso a internet saliente a
graph.facebook.com y vuelve a intentarlo.)

## Credenciales válidas, pero no llegan mensajes

1. Haz clic en **Test API Connection**. Si dice *"The WhatsApp Business
   Account is not subscribed to this app"*, vuelve a ingresar el token y
   guarda de nuevo — el guardado suscribe la WABA.
2. Revisa el aviso *Registration status*. *Not registered* significa que
   `/register` nunca tuvo éxito para este número; ingresa el PIN de dos
   pasos y guarda.
3. Haz clic en **Verify with Meta** para ver un desglose por verificación
   (`phone_metadata_ok`, `waba_subscribed_to_app`,
   `locally_marked_registered`).
4. Confirma el webhook en sí: en la App de Meta → WhatsApp →
   Configuration, la URL de callback debe ser
   `https://<tu host>/api/whatsapp/webhook`, el token de verificación
   debe coincidir con el guardado aquí y el campo `messages` debe estar
   suscrito. `META_APP_SECRET` en el entorno del servidor debe ser el
   secreto de *esa* app, o toda entrega se rechaza con un 401 antes de
   que wacrm la examine.

## Dónde vive el mapeo

* `src/lib/whatsapp/meta-error-explain.ts` — código → explicación, campo
  y lado 400/502. Puro y con pruebas unitarias; agrega ahí los códigos
  nuevos.
* `src/lib/whatsapp/waba-pairing.ts` — verificación del formato de id y
  de que el número pertenezca a la WABA.
* `src/app/api/whatsapp/config/route.ts` — el flujo de conexión. Todo
  fallo de Meta devuelve `{ error, meta: { code, subcode, fbtrace_id,
  step, field, message } }`.
