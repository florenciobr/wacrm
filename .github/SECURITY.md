# Política de Seguridad

Gracias por dedicar tiempo a revisar la seguridad de esta plantilla.

## Reportar una vulnerabilidad

**No abras un issue público en GitHub para errores de seguridad.** Los
issues públicos son indexados por los buscadores y los ven todos los forks
mucho antes de que llegue la corrección al repositorio original.

En su lugar, repórtalo de forma privada por una de estas vías:

- [GitHub Security Advisories](https://github.com/ArnasDon/wacrm/security/advisories/new)
  (preferida — mantiene la divulgación, la corrección y el CVE en un
  mismo lugar).
- Correo: `a.donauskas@hostinger.com` con `[CRM template security]` en el
  asunto.

Incluye, si puedes:

- Una descripción del problema y su impacto.
- Pasos de reproducción o una prueba de concepto.
- El commit o la versión con la que estás probando.
- Si quieres recibir crédito en la divulgación final (por defecto damos
  crédito con el nombre o alias que nos indiques, salvo que prefieras el
  anonimato).

## Qué esperar

- **Acuse de recibo** en un plazo de 72 horas.
- **Evaluación inicial** (gravedad, versiones afectadas, si existe una
  solución temporal) en una semana.
- **Corrección + divulgación coordinada** en un plazo proporcional a la
  gravedad. Los problemas críticos publican un parche en cuanto esté
  listo; los de gravedad media se agrupan con la siguiente versión.

## Alcance

Dentro del alcance:
- Cualquier cosa de este repositorio (`ArnasDon/wacrm`), incluidos los
  flujos de webhook y autenticación, el cifrado de tokens, las políticas
  RLS y los endpoints cron integrados.
- Las configuraciones por defecto incluidas en `docs/` — p. ej., si la
  guía de configuración deja un valor por defecto inseguro.

Fuera del alcance:
- Vulnerabilidades en Supabase, Next.js, Node.js u otras dependencias
  ajenas — repórtalas a sus responsables. Con gusto actualizaremos
  versiones si se solicita.
- Problemas que requieran un despliegue ya comprometido (p. ej., una
  clave service-role filtrada), salvo que amplíen el alcance del daño más
  allá del compromiso inicial.
- Ingeniería social, ataques físicos o servicios de terceros que tu fork
  agregue después del despliegue.

## Puerto seguro

La investigación realizada bajo esta política está autorizada. No
emprenderemos acciones legales contra quien:

- Haga un esfuerzo de buena fe por evitar la destrucción de datos,
  violaciones de privacidad o interrupciones del servicio.
- Nos dé un tiempo razonable para responder antes de cualquier
  divulgación pública.
- No explote el problema más allá de lo necesario para demostrarlo.

Gracias por ayudar a mantener segura esta plantilla (y sus forks).
