# SismoVE — Lo construido y lo aprendido

> Playbook reutilizable para **futuras apps de respuesta a crisis / humanitarias**.
> Resume qué es SismoVE, cómo está hecho, los patrones que valen la pena copiar y las
> trampas que ya pagamos. Pensado para arrancar la próxima app parecida sin repetir errores.

Actualizado: 2026-11-05 · App en producción: https://sismove-venezuela.vercel.app

---

## 1. Qué es SismoVE

PWA humanitaria para el terremoto doble de Venezuela (M7.2 + M7.5, jun-2026). Ayuda a la
gente a **buscar personas** (desaparecidos, encontrados/a salvo, ingresos a hospitales),
**ver refugios/albergues en un mapa**, **pedir ayuda** (SOS, rescate, logística),
encontrar **cobertura de internet**, **recursos de apoyo** (meditación, respiración) y
**centros de acopio** nacionales e internacionales. Bilingüe (ES/EN), con modos claro /
oscuro / estrés (accesibilidad).

Público objetivo: personas afectadas, familiares en el exterior, voluntarios y rescatistas.
El dueño es **no técnico**, valora diseño intuitivo y visual, y responde en español.

---

## 2. Arquitectura y stack (lo que funcionó)

- **Frontend: HTML/CSS/JS estático, sin framework.** Una SPA en `app.html` (~3000 líneas)
  con vistas conmutadas por `go(vista)` (muestra/oculta `<section class="view" id="v-…">`).
  Subpáginas estáticas propias: `calma.html`, `medita.html`, `replicas.html`, `visor3d.html`,
  `index.html`, `404.html`. CSS compartido en `shared.css` + sistema de tokens en
  `design-tokens.css`. **Ventaja:** cero build, despliegue instantáneo, fácil de razonar.
- **Backend: Vercel Edge Functions** (`api/*.ts`, `export const config = { runtime: 'edge' }`).
  Proxies y sincronizadores: `track`, `analytics`, `sync-drive`, `sync-fotos`,
  `sync-hospitales`, `drive-docs`, `reconecta`, `cifras`, `sismo-stats`, `reportar-persona`,
  `admin-update`, `admin-delete`, `notify`, `telegram/webhook`.
- **Datos: Supabase** (Postgres + PostgREST + RLS + Storage). El cliente lee con la
  **anon key** (pública, ya en el HTML); la **service_role** solo vive en los endpoints
  server-side. Tablas comunitarias: lectura + inserción anónimas, edición/borrado solo
  service_role vía `admin-*`.
- **Cron: GitHub Actions** (`.github/workflows/cron.yml`), 1×/hora con bucle interno de
  ~5 pasadas (≈50 min) para datos frescos sin solapar corridas.
- **Mapas: Leaflet + tiles CARTO** (`light_all`). Capas conmutables (`activeLayers` Set,
  `focusLayer`, `toggleLayer`/`addLayer`/`buildXxx`/`renderHub`). El visor 3D usa
  **MapLibre, NO Cesium** (Cesium congela la GPU del usuario).
- **i18n propio:** diccionario `I18N = {es:{…}, en:{…}}` + atributos `data-i18n`
  (`data-i18n-ph`, `data-i18n-html`) + `t()`/`applyI18n()`/`setLang()`. Idioma en
  `localStorage` (`sismove_lang`).
- **Temas:** `setTheme('light'|'dark'|'stress')` → clases `body.theme-dark`/`theme-stress`,
  guardado en `localStorage` (`sismove_theme`).
- **Despliegue:** `vercel --prod --yes` (los avisos `TS2580 Cannot find name 'process'` en
  las Edge Functions son **no fatales**: el deploy queda READY igual).

---

## 3. Funcionalidades construidas

- **Inicio** (`¿Qué necesitas?`): 9 tiles 3×3 (2 col en móvil) con iconos SVG + tonos por
  tile, cada uno con su acción (`go(...)`, abrir reporte o enlace externo).
- **Buscar personas:** buscador por nombre en `desaparecidos_reportes` (+ federado con Red
  Ayuda) y en `hospital_admisiones` (ingresos a hospitales).
- **Desaparecidos:** buscador arriba + reportar + cifras + portales externos + grupos por zona.
- **Refugios/albergues:** mapa con solo los albergues (coords geolocalizadas); al tocar uno,
  buscador de nombres filtrado a ESE refugio (¿está esta persona a salvo aquí?).
- **Mapa completo** (barra inferior): todas las capas (zonas afectadas, SOS, celdas, acopio,
  cobertura/internet, luz, logística).
- **Encontrados / a salvo:** 1.259 sobrevivientes de La Guaira importados del Drive.
- **Cobertura de internet:** puntos WiFi/Starlink de Reconecta Venezuela (proxy horario).
- **Recursos (apoyo):** meditación guiada (`/medita`) + más recursos (`/calma`).
- **SOS / Rescate / Logística:** enlazan a la plataforma externa Red de Esperanza.
- **Bot de Telegram** con `/estadisticas` (analítica explicada métrica por métrica).
- **Botón flotante de WhatsApp**, ticker de réplicas, centros de acopio nacionales/intl.

---

## 4. Flujos de datos (lo más valioso para reutilizar)

### 4.1 Lector universal Drive → Supabase (`api/sync-drive.ts`)
El dueño deja un archivo en una carpeta pública de Drive y el cron lo lee y lo lleva a la
tabla correcta. Patrón clave:
- **Ruteo por NOMBRE de archivo** → tabla destino (desaparecidos, hospital, acopio,
  cobertura, luz, logística, zonas, **sobrevivientes→encontrados**). Un `ADAPTERS[]` con
  `keywords` (regex) + `build(row, cell, ctx)`.
- Soporta **CSV, XLSX (parser ZIP+inflate propio, sin dependencias), JSON y Google Sheets**.
- **Mapeo flexible de encabezados** en español (`FIELD_SYNS`), detección de la fila de
  encabezado real, concatenación de columnas de nombre.
- **Seguridad de datos:** redacta cédula/documento **por valor** (no solo por encabezado),
  `contacto` solo formato telefónico, descarta filas basura/encabezados repetidos,
  `foto_url` solo de hosts confiables.
- **Modos:** *aditivo* (Supabase manda; solo agrega, no revierte ni borra — p.ej.
  desaparecidos/encontrados) vs *espejo* (refleja el archivo, pero no borra si la descarga
  parece truncada). Identidad por **ID de Drive** (estable ante rename).
- **Carpeta PROHIBIDA** excluida siempre (`DRIVE_EXCLUDE_IDS`): nunca se lista ni se lee.
- Si falta lat/lng, geocodifica por nombre con **Nominatim/OSM** (acotado por corrida).

### 4.2 Otros
- **OCR de fotos** (`sync-fotos`) para extraer info de imágenes nuevas del Drive.
- **Reconecta Venezuela** (`reconecta.ts`): su `/data/sites.json` exige cabecera `Referer`
  (anti-hotlink) y no da CORS → proxy server-side con `Referer`+`User-Agent`, cacheado,
  **con atribución visible** (no se saltan protecciones, solo se reenvía un JSON público).
- **Analítica anónima** (`track`/`analytics`): eventos `visit/view/search/report/bot` con
  ubicación aproximada por IP (cabeceras de Vercel, **nunca se guarda la IP**).

---

## 5. Aprendizajes reutilizables (patrones y trampas)

1. **Verifica siempre contra la base real, no de memoria.** Antes de afirmar cifras, consulta
   la fuente (Supabase REST, el endpoint, el archivo). Nada de números inventados; los datos
   demo van **claramente etiquetados** (`DEMO`).
2. **"Reporte enviado" ≠ reporte guardado.** El flujo de emergencias del Panel guardaba en
   **localStorage del navegador**, no en una base central → nadie recibía los reportes.
   *Lección:* comprueba que cada formulario **persiste en un store central** (tabla/endpoint),
   no solo en el dispositivo. Un toast de éxito no prueba nada.
3. **Supabase Storage tiene allowlist de MIME por bucket.** El bucket de fotos (`reportes`)
   rechaza audio. *Lección:* crea un bucket dedicado (`media`, público) con
   `allowed_mime_types` para audio/otros. URL pública: `…/storage/v1/object/public/<bucket>/<path>`.
4. **Geolocaliza una vez y fija coords cuando son pocas.** Para ~4 albergues: geocodificar con
   Nominatim (UA correcto) y **hardcodear** las coords en el código es más fiable que
   geocodificar en runtime. Documentar que añadir centros nuevos requiere añadir sus coords.
5. **Verificación sin capturas.** `preview_screenshot`/`snapshot` **congelan el PC** del dueño.
   Verificar leyendo el DOM/consola con `preview_eval` + `preview_console_logs`. Tras editar
   un artefacto publicado: **reload** y leer el estado, no una captura.
6. **Marca blanca neutra, sin carga política.** El rojo intenso se asocia a un partido →
   fondo del splash cambiado a hueso (`#F6F2EA`), manteniendo logo/acento. En apps
   humanitarias, cuidar que el color/branding no tome bando.
7. **i18n desde el principio.** Textos estáticos con `data-i18n`; los render dinámicos
   (resultados de búsqueda, tiles) se **re-traducen** al cambiar idioma (`renderNeeds()` en
   `setLang`). El diccionario es una sola fuente.
8. **Separar pantallas por intención.** "Buscar" (nombre + hospitales) ≠ "Desaparecidos"
   (reportar + portales) ≠ "Refugios" (mapa + a salvo). Un botón = una intención clara.
9. **Edge Functions:** avisos `TS2580 process` son ruido; el deploy queda READY. No bloquean.
10. **Cron robusto:** 1×/hora con bucle interno largo (≈50 min) + `concurrency` sin
    cancel-in-progress → datos frescos sin avalancha de corridas "cancelled".
11. **Explicar las métricas en el producto.** En `/estadisticas` y en el informe, cada número
    lleva una línea en lenguaje llano de qué significa. Un no-técnico debe entenderlo solo.
12. **Flujo de cambio:** editar → **verificar en preview** (eval/consola) → commit con
    identificador → push → `vercel --prod --yes` → confirmar producción. Siempre en ese orden.

---

## 6. Constraintes y seguridad (respetadas aquí, repetir en futuras)

- **Nunca leer/extraer** de la carpeta de Drive prohibida (`1dZ1jSXww…`), excluida en
  `sync-drive`/`sync-fotos`/`drive-docs`.
- **Secretos solo en `.env`/Vercel**, jamás en el repo. `service_role` solo server-side.
  Al usar `vercel env pull .env.vercel`, **borrarlo tras usarlo**.
- **No inventar datos** ni requisitos. Si no hay fuente, decirlo.
- **Redactar cédulas/documentos** en toda importación automática.
- **No saltar CAPTCHA / anti-bot.** Respetar `Referer`/atribución de las fuentes.

---

## 7. Deuda conocida / pendientes (para retomar si se sigue)

- **Emergencias del Panel sin base central:** el formulario "Pedir ayuda" guarda solo en
  localStorage; no existe tabla `reports`/`emergencias`. Lo real que SÍ se recoge:
  desaparecidos/encontrados (~1.836) y logística (~23). Decisión pendiente: arreglar el flujo,
  dar acceso a la data real, o quitar los DEMO del Panel.
- **Coords de albergues hardcodeadas** (3 puntos de La Guaira). Centros nuevos = añadir coords.
- **Vistas huérfanas** tras simplificar la navegación (donaciones, panel, etc. sin acceso
  desde la interfaz). Reconectar las que hagan falta.
- Pregunta abierta del dueño: **"¿hay casos reales reportados?"** (quedó pausada).

---

## 8. Archivos clave

```
app.html                  SPA principal (todas las vistas + JS)
index.html, 404.html      portada y error
calma.html / medita.html  recursos de apoyo (respiración / meditación)
replicas.html, visor3d.html  réplicas y visor 3D (MapLibre)
shared.css, design-tokens.css  estilos
api/*.ts                  Edge Functions (sync, proxies, bot, analítica)
supabase/*.sql            esquema + RLS (ejecutar en SQL Editor)
.github/workflows/cron.yml  sincronización horaria
ONBOARDING.md             guía para colaboradores (env, RLS, arranque)
informe-estadisticas.html informe de evolución de la analítica (snapshot)
```
