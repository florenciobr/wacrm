# Cómo usar esta plantilla

Este es un **repositorio plantilla**, no un producto colaborativo. El
flujo esperado es:

1. Haz un **fork** a tu propia cuenta u organización de GitHub.
2. **Despliega** el fork — consulta [`docs/`](./docs/README.md).
3. **Personaliza** tu fork. Cambia la marca, agrega las funciones que
   necesites, quita las que no, cambia de hosting, modifica el esquema.

**No** necesitas enviar cambios al repositorio original. Que tu fork
diverja es justamente la idea — el repositorio original es deliberadamente
opinado en cuanto a stack, UX y alcance, y tu fork es donde esas
opiniones pasan a ser tuyas.

## Haz un fork y ejecútalo

```bash
# 1. Fork en GitHub: https://github.com/ArnasDon/wacrm → Fork
# 2. Clona tu fork
git clone https://github.com/<tu-usuario>/wacrm.git
cd wacrm

cp .env.local.example .env.local   # completa credenciales de Supabase + Meta
npm install
npm run dev
```

La configuración completa (migraciones de Supabase, API de WhatsApp
Business, despliegue) está en [`docs/`](./docs/README.md).

## Mantener tu fork actualizado

Incorpora periódicamente las correcciones de errores y parches de
seguridad del repositorio original:

```bash
git remote add upstream https://github.com/ArnasDon/wacrm.git  # una sola vez
git fetch upstream
git checkout main
git merge upstream/main     # o: git rebase upstream/main
# Resuelve los conflictos (probablemente en áreas que personalizaste) y luego haz push
git push origin main
```

Si hiciste muchas personalizaciones locales, hacer rebase puede generar
conflictos cada vez que actualices. Fijarte a una etiqueta específica del
repositorio original y actualizar según tu calendario es una alternativa
válida.

## Reportar errores en la plantilla original

Si encuentras un error en el código original — no uno que introdujiste en
tu fork — por favor repórtalo con la plantilla de
[reporte de errores](https://github.com/ArnasDon/wacrm/issues/new?template=bug_report.yml).
Incluir el SHA del commit, el entorno de ejecución (Hostinger / Vercel /
local / otro) y los logs es lo que más acelera la corrección.

## Reportar problemas de seguridad

**No reportes problemas de seguridad de forma pública.** Sigue el proceso
privado descrito en [SECURITY.md](./.github/SECURITY.md).

## Pull requests al repositorio original

No es el flujo principal, pero son bienvenidos en casos específicos:

- **Correcciones de seguridad** — siempre bienvenidas; sigue primero
  SECURITY.md para la divulgación.
- **Correcciones de errores** acordes con la intención original (fallos,
  corrección funcional, errores de documentación, erratas) — se integran
  rápido.
- **Mejoras pequeñas** (accesibilidad, detalles evidentes de UX) —
  normalmente bienvenidas; abre antes un issue para confirmar que encajan.

Menos probables de integrarse:

- **Funciones nuevas.** El alcance de la plantilla es intencionalmente
  acotado. Una "gran idea para un CRM" suele ser una gran idea para *tu*
  CRM — es decir, tu fork — pero diluiría la plantilla para el siguiente
  usuario.
- **Cambios de stack** (otro ORM, otro kit de UI, otro proveedor de
  autenticación). Pertenecen a un fork, no al repositorio original.
- **Refactorizaciones opinadas** sin una motivación concreta de
  corrección o rendimiento.

Si envías un PR, aplican las reglas habituales:

- Crea tu rama desde el `main` más reciente (no hagas push a una rama ya
  fusionada — los commits quedan huérfanos).
- Ejecuta localmente `npm run typecheck` y `npm run format` antes.
- Completa la plantilla del PR, sobre todo el **Plan de pruebas**.
- Un cambio lógico por PR.
- La primera línea del mensaje de commit es imperativa y breve; el cuerpo
  explica el *porqué*, el diff muestra el *qué*.

Espera una revisión en unos pocos días. Los PR abiertos sin un issue
previo pueden cerrarse — abre primero el issue para alinear.

## Si mantienes un fork público

- Cambia la marca. El nombre "CRM Template for WhatsApp", el favicon y la
  URL `wacrm.tech` pertenecen al proyecto original; por favor reemplázalos
  por los tuyos antes de poner tu despliegue frente a usuarios.
- Conserva el archivo [`LICENSE`](./LICENSE) MIT — así es como los
  permisos de la plantilla viajan con el código. Una mención en una
  sección del `README` se agradece, pero no es obligatoria.
- Eres libre de relicenciar las adiciones a tu fork como quieras.

## Referencia del ciclo de desarrollo

Aunque nunca envíes un PR al repositorio original, estos son los scripts
que usarás en tu fork:

| Comando | Qué hace |
| --- | --- |
| `npm run dev` | Servidor de desarrollo con Turbopack en el puerto 3000. |
| `npm run build` | Build de producción. Next también ejecuta aquí su propio typecheck. |
| `npm run typecheck` | `tsc --noEmit`. Pasada rápida solo de TypeScript. |
| `npm run lint` | ESLint. |
| `npm run format` | Prettier en modo escritura. |
| `npm run format:check` | Prettier en modo solo verificación. Útil en CI. |

## Licencia

Esta plantilla es MIT ([`LICENSE`](./LICENSE)). Se asume que todo lo que
aportes al repositorio original también es MIT. Las adiciones de tu fork
son tuyas para licenciarlas como quieras.
