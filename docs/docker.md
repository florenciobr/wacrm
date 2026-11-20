# Ejecutar con Docker

El repositorio incluye un `Dockerfile` multi-etapa (salida standalone de
Next.js, se ejecuta como usuario no root) y un `docker-compose.yml` con un
único servicio `app`. Supabase es externo — apunta la app a tu proyecto de
Supabase alojado (o autoalojado) mediante variables de entorno; no se
incluye ningún contenedor de base de datos.

## Inicio rápido

1. Copia la plantilla de entorno y complétala:

   ```bash
   cp .env.local.example .env.local
   ```

2. Compila y arranca (la opción `--env-file` es obligatoria — Compose solo
   lee `.env` por defecto para la sustitución de `${VAR}`, y este proyecto
   guarda su configuración en `.env.local`):

   ```bash
   docker compose --env-file .env.local up --build -d
   ```

3. La app se sirve en [http://localhost:3000](http://localhost:3000)
   (publícala en otro puerto con `HOST_PORT=8080` en `.env.local`).

> Usa `HOST_PORT`, no `PORT`, para cambiar el puerto publicado. `PORT` es
> el puerto en el que escucha el servidor _dentro_ del contenedor, y
> `env_file` lo inyectaría ahí — dejando la app en un puerto al que no
> apuntan ni el mapeo ni el healthcheck. Compose lo fija en 3000 por esa
> razón.

## Variables de compilación vs. de ejecución

- Las variables `NEXT_PUBLIC_*` se **incrustan en el bundle del cliente
  durante la compilación**. `docker-compose.yml` las pasa como build args
  de Docker. Si cambias alguna, recompila:
  `docker compose --env-file .env.local up --build -d`. Esto incluye
  `NEXT_PUBLIC_APP_LOCALE` (`en | ko | pt | es`), por lo que el idioma de
  la interfaz queda fijo por imagen.
- Todo lo demás (`SUPABASE_SERVICE_ROLE_KEY`, `ENCRYPTION_KEY`,
  `META_APP_SECRET`, …) se lee en **tiempo de ejecución** desde
  `.env.local` mediante `env_file` y nunca se incorpora a la imagen — se
  puede cambiar con solo reiniciar el contenedor.

## Docker simple (sin Compose)

```bash
docker build \
  --build-arg NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co \
  --build-arg NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key \
  -t wacrm .

docker run -d --env-file .env.local -e PORT=3000 -p 3000:3000 wacrm
```

## Notas

- El contenedor **no** ejecuta las migraciones de base de datos de
  `supabase/` — aplícalas con la CLI de Supabase como se describe en el
  README.
- Los adjuntos recibidos se copian al bucket `chat-media` de Supabase
  Storage, porque Meta elimina los archivos multimedia aproximadamente 30
  días después de recibirlos y la copia es lo único que sobrevive a ese
  plazo. Crece con el volumen entrante, así que conviene vigilar la cuota
  de almacenamiento de tu proyecto. Desactívalo por cuenta en Ajustes →
  WhatsApp → Almacenamiento de adjuntos; los adjuntos recibidos mientras
  está desactivado dejan de poder verse cuando Meta los elimina. Los
  archivos de más de 16 MB (el límite del bucket) nunca se copian.
- Nada dentro del contenedor está programado. Si usas pasos de Espera o
  flujos de automatización, apunta un programador externo a
  `GET /api/automations/cron` y `GET /api/flows/cron` de este despliegue,
  enviando el secreto compartido en el encabezado `x-cron-secret`
  (`AUTOMATION_CRON_SECRET`, consulta `.env.local.example`). Ambos
  devuelven 503 hasta que se define esa variable.
