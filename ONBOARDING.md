# SismoVE · Guía para colaborar 🇻🇪

App humanitaria por el terremoto de Venezuela (doblete M7.2 + M7.5, Yaracuy, 24-jun-2026).
Producción: **https://sismove-venezuela.vercel.app** · App/mapa: `/app`.

Esta guía es para que **otra persona** se ponga a trabajar en el proyecto sin depender de una sola persona. Hay **3 formas de colaborar**, de menos a más acceso:

---

## 1) Colaborar SIN código (voluntarios) — no necesita nada técnico

Todo esto alimenta la app en automático:

- **Reportar en la app** (`/app`): desaparecidos, niños, SOS en sitio, zonas afectadas, logística (qué falta), cobertura/luz, grupos por zona.
- **Bot de Telegram**: buscar personas, ver centros de acopio, novedades. (`/start` en el bot.)
- **Dejar archivos en el Google Drive compartido**: el sistema los lee solo (cada ~hora):
  - **Sheets / CSV / XLSX** → se rutean por el nombre del archivo a la tabla correcta (desaparecidos, hospital, acopio, logística…).
  - **Imágenes / PDF** → OCR con IA que clasifica y enruta (carteles de desaparecidos/rescatados, listas de hospital, acopio, daños, necesidades, grupos).
  - Para que una carpeta se lea: **Compartir → "Cualquiera con el enlace → Lector"** y añadirla a `DRIVE_FOLDER_IDS`.

> El dueño debe dar acceso (Lector/Editor) a las carpetas del Drive a quien vaya a subir archivos.

---

## 2) Colaborar CON código (desarrolladores)

### Stack
- **Frontend**: HTML/CSS/JS puro, sin framework. `index.html` (portada) + `app.html` (dashboard/mapa, Leaflet) + subpáginas (`replicas.html`, etc.). Es prácticamente una PWA en pocos archivos.
- **Backend**: Vercel **Edge Functions** en `api/*.ts` (`export const config = { runtime: 'edge' }`).
- **Datos**: **Supabase** (PostgREST + RLS). El cliente usa la `anon key`; el servidor usa `service_role` (nunca en el cliente).
- **Bot**: Telegram + **Gemini** (`api/telegram/webhook.ts`).
- **Cron**: GitHub Actions (`.github/workflows/cron.yml`) que pinguea los endpoints `/api/*`.

### Empezar en local
```bash
git clone https://github.com/fredparedes58-ui/sismove-venezuela.git
cd sismove-venezuela
node scraper/scrape.mjs      # genera data/feed.json/feed.js con datos reales (opcional)
npm run serve                # sirve los archivos estáticos (scripts/serve.mjs)
# abre index.html / app.html
```
> El frontend lee Supabase directamente con la `anon key` (ya pública en el HTML), así que el mapa, buscador y acopio funcionan en local sin secretos. Los endpoints `/api/*` solo corren desplegados en Vercel (o con `vercel dev`).

### Flujo de trabajo
1. Crea una rama, haz cambios, abre un **Pull Request** a `main`.
2. Verifica: `node -e` para chequear sintaxis del JS embebido, o `vercel dev` para los endpoints.
3. Deploy a producción: `vercel --prod --yes` (o auto-deploy si Vercel está enlazado al repo).
4. Los cambios en `api/*.ts` requieren **deploy a Vercel**; los de `cron.yml` requieren **push a GitHub**.

### Mapa de archivos clave
| Archivo | Qué hace |
|---|---|
| `app.html` | Dashboard: mapa (Leaflet), buscador, reportes, SOS, acopio, grupos |
| `index.html` | Portada + cifras de la emergencia |
| `api/sync-drive.ts` | Importa Sheets/CSV/XLSX del Drive → tablas |
| `api/sync-fotos.ts` | OCR (Gemini) de imágenes/PDF del Drive → enruta a la tabla correcta |
| `api/sync-hospitales.ts` | Sincroniza ingresos hospitalarios |
| `api/cifras.ts` | Cifras de la portada (desde la BD pública de redayuda) |
| `api/replicas.ts` | Réplicas sísmicas (USGS) |
| `api/notify.ts` | Avisos a suscriptores del bot cuando hay datos nuevos |
| `api/telegram/webhook.ts` | Bot de Telegram (+ `/estadisticas` admin) |
| `api/track.ts` / `api/analytics.ts` | Analítica anónima de uso |
| `api/admin-update.ts` / `api/admin-delete.ts` | Edición/borrado admin (lista blanca de tablas) |
| `supabase/*.sql` | Esquemas (ejecutar en Supabase SQL Editor) |

### Variables de entorno (Vercel → Settings → Environment Variables)
**Secretas (pedírselas al dueño; nunca en el repo):**
- `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` — base de datos (service_role solo server-side)
- `SCRAPER_WEBHOOK_SECRET` — clave del cron para los endpoints `/api/*`
- `SCRAPER_WEBHOOK_URL` — destino del scraper
- `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET` — bot
- `ADMIN_KEY` — edición/borrado admin desde la app
- `ADMIN_CHAT_IDS` — chats de Telegram que pueden ver `/estadisticas`
- `GEMINI_API_KEY` — OCR + bot (free tier)
- `OPENCELLID_API_KEY` — torres de cobertura (opcional)

**De configuración (no secretas):**
- `DRIVE_FOLDER_IDS` — carpetas del Drive que se rastrean (coma-separadas)
- `DRIVE_EXCLUDE_IDS` — carpetas PROHIBIDAS (nunca se leen). **Ver reglas abajo.**
- `GEMINI_MODEL` — opcional (def `gemini-2.5-flash`)

### Supabase
Ejecutar los `supabase/*.sql` en el SQL Editor (empezar por `schema.sql`; los `schema_*.sql` añaden tablas/columnas: sos, grupos, logística, zonas, analítica, etc.). Todas las tablas comunitarias tienen RLS: lectura + inserción anónimas; **edición/borrado solo service_role** (vía `admin-update`/`admin-delete`).

---

## 3) Colaborar en OPERACIONES (admin)

- **Vercel**: el dueño invita al equipo del proyecto (`fred-paredes-projects/sismove-venezuela`) → deploys y env.
- **GitHub**: el dueño añade colaboradores al repo `fredparedes58-ui/sismove-venezuela` (Settings → Collaborators).
- **Supabase / Telegram / Drive / Gemini**: accesos que solo el dueño puede otorgar.
- **`/estadisticas`** del bot: solo para los chat IDs en `ADMIN_CHAT_IDS`.

---

## ⚠️ Reglas permanentes del proyecto (no romper)
- **NUNCA** leer ni extraer de la carpeta de Drive `1dZ1jSXwwzkqPa7F4N-9xFWxuELfiqXlH` (ya excluida en `DRIVE_EXCLUDE_IDS`). No quitarla.
- **No inventar datos** (cifras, nombres, direcciones, teléfonos): solo datos reales de las fuentes.
- **Cédulas y documentos de identidad NO se almacenan** en la importación automática (se redactan).
- **Secretos solo en `.env`/Vercel**, jamás en el repo. `service_role` solo server-side.
- **No saltar CAPTCHA / anti-bot** de fuentes externas (p. ej. desaparecidosterremotovenezuela.com solo se enlaza).
- Responder al usuario en **español**.

## Pasos para que el dueño habilite a alguien nuevo
1. **GitHub** → repo → Settings → Collaborators → invitar por usuario.
2. **Vercel** → Project → Settings → Members → invitar (para deploy/env).
3. **Supabase** → Project → Settings → Team (si necesita tocar la BD).
4. Compartir las **carpetas del Drive** que deba leer/editar.
5. Pasarle esta guía (o el enlace de onboarding para abrirla en Claude Code).
