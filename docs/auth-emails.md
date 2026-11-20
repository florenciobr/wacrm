# Correos de autenticación: enlaces de confirmación y restablecimiento de contraseña

wacrm usa Supabase Auth para el registro, el inicio de sesión y el
restablecimiento de contraseña. Dos de esos flujos envían un correo con
un enlace, y ese enlace tiene que volver a **tu** despliegue. Esta página
explica cómo lo solicita la app, qué hay que configurar en Supabase para
que lo respete y cómo se ven los síntomas cuando no lo hace.

## Qué hace la app

A cada enlace enviado por correo se le pide que regrese al origen propio
de la app, en `/auth/callback`, que intercambia el enlace por una sesión y
luego redirige a la página que necesita el flujo:

| Flujo | Se inicia desde | El enlace regresa a |
|---|---|---|
| Confirmación de correo tras el registro | `/signup` | `/auth/callback?next=/dashboard` (o `/join/<token>` al registrarse desde una invitación) |
| Restablecimiento de contraseña | `/forgot-password` | `/auth/callback?next=/reset-password` |

"El origen propio de la app" es el origen desde el que el navegador cargó
la página (`window.location.origin`) — `https://crm.example.com` en
producción, `http://localhost:3000` en desarrollo local. No interviene
ninguna variable de entorno.

`/auth/callback` acepta las dos formas de enlace que Supabase puede
generar — el `?code=` (PKCE) por defecto y `?token_hash=&type=` de una
plantilla personalizada —, valida `next` para que solo pueda apuntar a
una ruta del mismo origen y redirige con un `Location` *relativo*, de modo
que funciona sin cambios detrás de Cloudflare Tunnel, nginx, Hostinger,
Vercel o cualquier otro proxy.

## Qué necesita Supabase

Supabase solo respeta una redirección solicitada cuando coincide con su
lista de permitidos **Redirect URLs**. Si no, **vuelve silenciosamente a
la Site URL**, cuyo valor por defecto es `http://localhost:3000`. Ese
retroceso es toda la explicación de "el enlace de confirmación apunta a
localhost".

### Supabase alojado (supabase.com)

Dashboard → **Authentication → URL Configuration**:

1. **Site URL** — tu origen de producción, p. ej. `https://crm.example.com`.
2. **Redirect URLs** — agrega un comodín para cada origen en el que corra
   la app:

   ```text
   https://crm.example.com/**
   http://localhost:3000/**
   ```

   El `/**` es importante: la redirección de la app lleva una ruta y una
   cadena de consulta (`/auth/callback?next=…`), que una entrada de solo
   el origen no cubre.

Los cambios surten efecto de inmediato, pero un correo que ya se envió
conserva el enlace con el que salió — solicita uno nuevo para probar.

### Supabase autoalojado (Docker)

La página de Studio en **Authentication → URL Configuration no guarda los
ajustes** en una instalación autoalojada; el servicio de autenticación
(GoTrue) los lee de su entorno. En el `.env` junto a tu
`docker-compose.yml`:

```env
SITE_URL=https://crm.example.com
ADDITIONAL_REDIRECT_URLS=https://crm.example.com/**,http://localhost:3000/**
```

(Las configuraciones más antiguas las llaman `GOTRUE_SITE_URL` y
`GOTRUE_URI_ALLOW_LIST`.) Reinicia después el contenedor de autenticación.
Una instalación autoalojada cuyo `.env` aún tiene el `SITE_URL=http://localhost:3000`
de ejemplo produce exactamente el síntoma del
[issue #595](https://github.com/ArnasDon/wacrm/issues/595).

### Supabase CLI local (`supabase start`)

El `supabase/config.toml` de este repositorio existe solo para el job de
CI de migraciones y deliberadamente no incluye una sección `[auth]`. Si
ejecutas el stack local, agrega:

```toml
[auth]
site_url = "http://localhost:3000"
additional_redirect_urls = ["http://localhost:3000/**"]
```

## Abrir el enlace en otro dispositivo

El flujo por defecto es PKCE: el navegador que *solicitó* el correo guarda
en una cookie un verificador de código de un solo uso, y `/auth/callback`
lo necesita para completar el intercambio. Si abres el enlace en otro
dispositivo o navegador, el callback termina en `/login` con un mensaje
que dice que no se pudo usar el enlace. Dos cosas a tener en cuenta:

- **La confirmación del correo sí se completó.** Supabase verificó la
  dirección antes de redirigir; la persona solo tiene que iniciar sesión.
- **El restablecimiento de contraseña debe terminarse en el mismo
  navegador** desde el que se solicitó — o solicita un enlace nuevo ahí.

Para que los enlaces no dependan del dispositivo, cambia las plantillas de
correo de Supabase (**Authentication → Email Templates**) a la forma con
token hash, que `/auth/callback` también acepta:

```html
<!-- Confirmar registro -->
<a href="{{ .SiteURL }}/auth/callback?token_hash={{ .TokenHash }}&type=email&next=/dashboard">Confirma tu correo</a>

<!-- Restablecer contraseña -->
<a href="{{ .SiteURL }}/auth/callback?token_hash={{ .TokenHash }}&type=recovery&next=/reset-password">Restablece tu contraseña</a>
```

Estos enlaces se construyen a partir de la **Site URL**, así que ese
ajuste debe ser tu origen de producción.

## Síntomas → causas

| Lo que ves | Causa probable |
|---|---|
| El enlace del correo es `http://localhost:3000…` en producción | La Site URL sigue en su valor por defecto y/o falta tu origen en Redirect URLs (ver arriba). En autoalojado: `SITE_URL` en el `.env` de Docker. |
| El enlace apunta al dominio correcto pero da 404 | Tienes un build anterior al que agregó `/auth/callback` y `/reset-password`. Actualiza tu fork. |
| `/login` dice que el enlace expiró | Los enlaces son de un solo uso y tienen tiempo limitado (por defecto en Supabase: 1 hora para recuperación). Solicita uno nuevo. |
| `/login` dice que no se pudo usar el enlace | Se abrió en un navegador distinto al que lo solicitó (PKCE), o ya se usó. Consulta la sección anterior. |
| El enlace funciona en localhost pero no a través de un túnel/proxy | No debería pasar — el callback redirige con un `Location` relativo. Verifica que tu proxy reenvíe la ruta `/auth/callback` y su cadena de consulta sin modificar. |

## Relacionado

- Los enlaces de invitación (`/join/<token>`) los genera la app, no
  Supabase — consulta `NEXT_PUBLIC_SITE_URL` y `ALLOWED_INVITE_HOSTS` en
  `.env.local.example`.
- El inicio de sesión (`/login`) nunca involucra un enlace por correo.
