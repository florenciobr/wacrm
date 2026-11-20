# wacrm — Plantilla de CRM para WhatsApp

> Plantilla de CRM autoalojable para WhatsApp® — bandeja compartida,
> contactos, embudos de venta, difusiones y automatizaciones sin código.
> Haz un fork, ponle tu marca, aloja tu propia instancia.

<p align="center">
  <a href="https://www.hostinger.com/web-apps-hosting?REFERRALCODE=WACRMHOST">
    <img src="./.github/assets/hostinger-deploy.png" alt="Lanza tu app Node.js con un clic — Despliega en Hostinger" width="900">
  </a>
</p>

[![License: MIT](https://img.shields.io/badge/License-MIT-violet.svg)](./LICENSE)
[![CI](https://github.com/ArnasDon/wacrm/actions/workflows/ci.yml/badge.svg)](https://github.com/ArnasDon/wacrm/actions/workflows/ci.yml)
[![Next.js 16](https://img.shields.io/badge/Next.js-16-black?logo=nextdotjs)](https://nextjs.org)
[![Supabase](https://img.shields.io/badge/Supabase-Postgres%20%2B%20Auth-3ecf8e?logo=supabase)](https://supabase.com)
[![Stars](https://img.shields.io/github/stars/ArnasDon/wacrm?style=social)](https://github.com/ArnasDon/wacrm/stargazers)

El sitio de marketing y la documentación de autoalojamiento viven en un
repositorio aparte:
[ArnasDon/wacrm-site](https://github.com/ArnasDon/wacrm-site)
([wacrm.tech](https://wacrm.tech)). Este repositorio es el producto:
clónalo o haz un fork para ejecutar tu propio CRM.

## Lo que incluye de fábrica

- **Bandeja compartida** sobre la API oficial de WhatsApp Business —
  varios agentes atendiendo un mismo número, con asignación por
  conversación, estado y notas.
- **Contactos + etiquetas + campos personalizados**, importación CSV,
  deduplicación.
- **Embudos de venta** (Kanban) con negocios vinculados a conversaciones.
- **Difusiones** con plantillas aprobadas por Meta, seguimiento de
  entrega y lectura, sustitución de variables por destinatario.
- **Automatizaciones sin código** — disparadores por mensajes entrantes,
  contactos nuevos, palabras clave o programación; ramas condicionales,
  esperas, etiquetas, webhooks. Constructor visual.
- **Asistente de respuestas con IA** — trae tu propia clave de OpenAI o
  Anthropic (se guarda cifrada; sin costo de IA por usuario, tus datos
  siguen siendo tuyos). Respuestas redactadas por IA con un clic en la
  bandeja, más un bot de respuesta automática opcional con límite por
  conversación y traspaso limpio a una persona. Agrega una **base de
  conocimiento** (preguntas frecuentes, políticas, documentación de
  producto) y responderá con tu propio contenido — recuperación híbrida
  (búsqueda de texto completo de Postgres, o pgvector semántico cuando
  hay una clave de embeddings).
- **Panel en tiempo real** — tiempos de respuesta, volumen diario, valor
  del embudo, feed de actividad entre módulos.
- **Cuentas de equipo** — invita compañeros por enlace, acceso por roles
  (propietario / administrador / agente / observador), transferencia de
  propiedad. Cada instalación está acotada a una cuenta, así que una
  bandeja compartida puede ser atendida por todo un equipo. El uso
  individual sigue siendo de un solo usuario y sin configuración.
- **Gestión de cuenta** — correo, contraseña, avatar, cierre de sesión
  global.
- **API REST pública** (`/api/v1`) con claves de API acotadas y
  revocables — construye tus propias automatizaciones sobre tu CRM.
  Consulta [docs/public-api.md](./docs/public-api.md).
- **Servidor MCP** — controla tu CRM desde Claude, Cursor y otros
  asistentes de IA mediante el [Model Context Protocol](https://modelcontextprotocol.io).
  Solo lectura por defecto, escritura opcional. Consulta
  [docs/mcp.md](./docs/mcp.md) (servidor en [`mcp-server/`](./mcp-server)).

## ¿Por qué hacer un fork?

Esto es una **plantilla**, no un producto. Al hacer un fork obtienes:

- **Propiedad total** — tu código, tu proyecto de Supabase, tu dominio,
  tus datos. Sin dependencia de un SaaS, sin precios por usuario, sin
  ceremonias de confianza.
- **Personalización total** — agrega los campos que tu equipo necesita,
  quita los módulos que no, rediseña lo que quieras. El stack es
  aburrido a propósito (Next.js + Supabase + Tailwind) para que la curva
  de aprendizaje sea corta.
- **Cero operaciones para empezar** — [Hostinger](https://www.hostinger.com/web-apps-hosting?REFERRALCODE=WACRMHOST)
  Managed Node.js despliega un fork en unos cuantos clics. Sin Docker,
  sin Kubernetes, sin equipo de infraestructura.
  ([Ver más abajo ↓](#-despliega-en-hostinger-recomendado))
- **Primitivas de seguridad reales** — cifrado de tokens (AES-256-GCM),
  RLS en cada tabla, webhooks verificados con HMAC, CSP, límite de
  peticiones, typecheck/build en CI en cada PR.

No es un framework. No es un SDK. Es un CRM concreto y funcional que
puedes levantar en una tarde y hacer tuyo.

## Inicio rápido

```bash
# Primero haz un fork en GitHub: https://github.com/ArnasDon/wacrm → Fork
git clone https://github.com/<tu-usuario>/wacrm.git
cd wacrm
npm install
cp .env.local.example .env.local   # completa credenciales de Supabase + Meta
npm run dev
```

Abre <http://localhost:3000>. Se te redirigirá a `/login` (o a
`/dashboard` si ya iniciaste sesión).

La interfaz viene en inglés, coreano, portugués de Brasil y español —
define `NEXT_PUBLIC_APP_LOCALE` como `en`, `ko`, `pt` o `es` en
`.env.local` (los catálogos están en `messages/`).

¿Prefieres contenedores? Consulta [docs/docker.md](./docs/docker.md)
para la configuración con Dockerfile y Docker Compose.

## 🚀 Despliega en Hostinger (recomendado)

<p align="center">
  <a href="https://www.hostinger.com/web-apps-hosting?REFERRALCODE=WACRMHOST">
    <img src="./.github/assets/hostinger-deploy.png" alt="Lanza tu app Node.js con un clic — Despliega en Hostinger" width="1000">
  </a>
</p>
<p align="center">
  <a href="https://wacrm.tech/docs/deployment-hostinger">
    <img src="https://img.shields.io/badge/Gu%C3%ADa_paso_a_paso-wacrm.tech%2Fdocs-111?style=for-the-badge" alt="Guía paso a paso" height="44">
  </a>
</p>

**wacrm está hecho para ejecutarse en [Hostinger](https://www.hostinger.com/web-apps-hosting?REFERRALCODE=WACRMHOST).**
Es el camino que probamos, documentamos y recomendamos — y la forma más
rápida de poner en producción un CRM de nivel profesional sin tener un
VPS ni un clúster de Kubernetes.

### ¿Por qué Hostinger?

| | |
|---|---|
| **Despliegue Git en un clic** | Conecta tu fork, haz push a `main` y Hostinger lo compila y lo publica. Sin SSH, sin Docker, sin CI que configurar — el propio `main` de este repositorio se despliega así. |
| **Node.js administrado** | Next.js 16 (App Router, server actions, ISR) funciona de inmediato en los planes compartidos [Premium, Business y Cloud](https://www.hostinger.com/web-apps-hosting?REFERRALCODE=WACRMHOST). No administras versiones de Node, procesos ni proxies inversos. |
| **SSL gratis + dominio gratis** | Let's Encrypt automático en tu dominio propio (o uno gratuito incluido en los planes anuales). HTTPS viene activado por defecto — requisito del webhook de WhatsApp Business. |
| **CDN global + LiteSpeed** | Recursos estáticos en caché en el borde, rutas dinámicas servidas desde LiteSpeed. Paneles ágiles de fábrica, sin configurar Cloudflare. |
| **Variables de entorno + logs en hPanel** | Define `SUPABASE_*`, `WHATSAPP_*` y `ENCRYPTION_KEY` desde el panel — sin `.env` en el servidor. Logs de la aplicación en vivo en la misma interfaz. |
| **Protección DDoS + copias diarias** | Incluidas, sin complementos. El endpoint del webhook es un objetivo público — la protección en el borde importa. |
| **Más barato que un VPS** | Los planes empiezan en unos pocos dólares al mes — un orden de magnitud menos que un hosting Node.js administrado comparable, y no pagas extra por la base de datos (eso es Supabase). |
| **Soporte humano 24/7** | Chat en vivo en más de 20 idiomas — útil cuando tu CRM es la herramienta con la que tu equipo habla con los clientes. |

### La versión de 60 segundos

1. Haz un **fork** de este repositorio en GitHub.
2. En **hPanel → Websites → Create**, elige **Node.js** y conecta tu
   fork.
3. Pega tus variables de entorno de Supabase + Meta en hPanel.
4. Haz push a `main`. Hostinger lo compila y lo sirve. Listo.

Guía completa con capturas de pantalla:
**[wacrm.tech/docs/deployment-hostinger](https://wacrm.tech/docs/deployment-hostinger)**.

> _Nota: wacrm tiene licencia MIT y corre en cualquier lugar donde corra
> Node.js (Vercel, Railway, tu propio VPS). Hostinger es recomendado, no
> obligatorio._

## Documentación

La documentación completa de autoalojamiento — migraciones de Supabase,
configuración de la API de WhatsApp Business y despliegue en producción —
está en **[wacrm.tech/docs](https://wacrm.tech/docs)**
(código fuente: [ArnasDon/wacrm-site](https://github.com/ArnasDon/wacrm-site)).

Páginas principales:
- [Primeros pasos](https://wacrm.tech/docs/getting-started)
- [Configuración de Supabase](https://wacrm.tech/docs/supabase-setup)
- [Configuración de WhatsApp](https://wacrm.tech/docs/whatsapp-setup)
- [Variables de entorno](https://wacrm.tech/docs/environment-variables)
- [Despliegue en Hostinger](https://wacrm.tech/docs/deployment-hostinger)
- [Arquitectura](https://wacrm.tech/docs/architecture)
- [Solución de problemas](https://wacrm.tech/docs/troubleshooting)
- [Solución de problemas de conexión con WhatsApp](./docs/whatsapp-connection-troubleshooting.md)
  — qué significa cada error de "Guardar configuración" y el código de
  Meta / trace id que debes citar al soporte de Meta
- [Varias WABA en un mismo despliegue](./docs/multi-waba.md) — una app
  de Meta o varias; cómo `META_APP_SECRET` acepta una lista separada por
  comas
- [Correos de autenticación](./docs/auth-emails.md) — qué debe permitir
  Supabase para que los enlaces de confirmación y de restablecimiento de
  contraseña regresen a *tu* dominio en lugar de `localhost:3000`, y
  cómo los maneja `/auth/callback`

## Stack

- **App** — Next.js 16 (App Router), React 19, TypeScript, Tailwind v4.
- **Datos** — Supabase (Postgres + Auth + Storage + RLS).
- **WhatsApp** — Meta Cloud API (API oficial de WhatsApp Business).

## Contribuir

Esto es una plantilla, no un producto colaborativo — el flujo esperado
es fork → personalizar → desplegar, **no** contribuir al repositorio
original. Los reportes de errores y de seguridad son bienvenidos; los PR
de nuevas funciones suelen pertenecer a tu fork y no a este repositorio.
Detalles en [`CONTRIBUTING.md`](./CONTRIBUTING.md) y
[`.github/SECURITY.md`](./.github/SECURITY.md).

## Licencia

[MIT](./LICENSE). Haz un fork, ponle tu marca, aloja tu propia instancia.
