## Why

YouTube no ofrece doblaje al español, y su traducción automática produce un literal que se pasa de tiempo: el español necesita ~1,25× las palabras del inglés, así que la línea traducida se solapa con la siguiente. Además, obtener la pista de subtítulos completa de un vídeo por vía programada está bloqueado: el endpoint `/api/timedtext` exige un *proof-of-origin token* (PO token) y responde HTTP 200 con el cuerpo vacío, por lo que un `fetch` directo no devuelve nada.

La oportunidad es una extensión de navegador que **hereda la petición ya atestada del reproductor** (sin gestionar ningún token), adapta la pista con un LLM bajo un **presupuesto de palabras por línea** derivado de la duración real de cada cue, y la reproduce sincronizada. El vídeo original no se modifica: la extensión solo observa, intercepta y superpone.

## What Changes

- **Nuevo `wxt.config.ts` con el manifiesto real**: `matches: ['*://*.youtube.com/*']`, `host_permissions` para `generativelanguage.googleapis.com` y `*.youtube.com`, y `web_accessible_resources` declarando el script de main world (WXT no los auto-registra).
- **Nuevo entrypoint `yt-main-world.ts`** (unlisted script, main world) que intercepta `fetch` y `XMLHttpRequest` sobre `/api/timedtext`, **parchea la URL en vuelo** para forzar `fmt=json3`, y expone el resultado vía `CustomEvent`. Expone además el acceso al player (`getPlayerResponse`, `setVolume`, `setOption`).
- **Nuevo entrypoint `youtube.content.ts`** (isolated world, `run_at: 'document_start'`) que orquesta la extracción, el reloj de reproducción, la cola de locución y la UI in-page.
- **Reemplazo de `entrypoints/content.ts`** (stub que hace match con `*://*.google.com/*`) por el content script de YouTube.
- **Reescritura de `entrypoints/background.ts`**: proxy a la API de Gemini (la API key nunca sale del service worker), caché por `videoId` y reintentos.
- **Reescritura del popup SolidJS**: formulario de API key (persistida en `storage.local`, nunca expuesta a la página), idioma destino, motor TTS, voz, nivel de compresión y estado de la traducción.
- **Cascada de 4 rutas de extracción** con escalada automática y **verificador de completitud** contra `videoDetails.lengthSeconds`, de modo que nunca se acepte en silencio una pista truncada.
- **Parsers de subtítulos** para `json3`, `srv3`, `srv1` y `vtt` (el endpoint **no** devuelve SRT) + limpieza de entidades HTML, tags de formato y timestamps inline.
- **Pipeline de adaptación** con `gemini-3.5-flash-lite` contra la Interactions API (`POST /v1beta/interactions`) y *structured output* (`response_format` con JSON Schema estándar), presupuesto de palabras por cue (`floor(dur × 2,9 × 0,92)`), chunking progresivo en ventanas de 120 s con look-ahead de 45 s, validación en cliente, reintento y fallback al texto original.
- **Motor de sincronización** con `requestVideoFrameCallback` y puntero monótono de cues (nunca igualdad de timestamps), con detección de seek, cancelación de locución en curso y retimificación de las líneas.
- **Overlay in-page** (shadow root) con el estado del pipeline y los subtítulos traducidos sincronizados.
- **Restricción de alcance**: este cambio entrega el pipeline completo hasta los **subtítulos traducidos y sincronizados**. La locución con TTS (motor navegador y motor Gemini) y el pulido de UX quedan para cambios posteriores, listados en *Fuera de alcance*.

## Capabilities

### New Capabilities
- `caption-extraction`: obtener la pista de subtítulos **completa** de un vídeo de YouTube, reutilizando la petición atestada del reproductor, conParsers multi-formato, limpieza, verificación de completitud, cascada de fallbacks y caché por `videoId`.
- `subtitle-adaptation`: adaptar la pista original al idioma destino bajo un presupuesto de palabras por línea, usando el LLM con structured output, chunking progresivo, validación en cliente y degradación segura al texto original.
- `synced-subtitle-playback`: reproducir la pista adaptada sincronizada con el vídeo mediante un reloj de cues con puntero monótono, y superponerla en la página con una UI de estado.
- `extension-configuration`: manifiesto y permisos correctos, persistencia tipada de la configuración y de la API key del usuario, popup de configuración y comunicación segura entre main world e isolated world.

### Modified Capabilities
<!-- Ninguna: openspec/specs/ está vacío, no hay specs existentes que modificar. -->

## Impact

**Código existente a modificar**
- `wxt.config.ts` — se añade el bloque `manifest` (permisos, `host_permissions`, `web_accessible_resources`).
- `entrypoints/content.ts` — se elimina (stub de `*.google.com`) y se sustituye por `entrypoints/youtube.content.ts`.
- `entrypoints/background.ts` — pasa de stub a proxy de Gemini + caché.
- `entrypoints/popup/App.tsx`, `index.html`, `App.css`, `style.css` — se sustituye el contador de demo por el formulario de configuración.

**Código nuevo**
- `entrypoints/yt-main-world.ts`, `entrypoints/youtube.content.ts`.
- `composables/captions/{extract,parse-json3,parse-xml,clean,verify}.ts`.
- `composables/ai/{translate,prompt}.ts`, `composables/budget.ts`.
- `composables/player/{clock,controller}.ts`.
- `utils/{types,storage}.ts`, `utils/bridge.ts`.

**Dependencias**
- Ninguna nueva. Se usan `wxt` 0.21.4, `solid-js` y `browser` (ya disponibles). La API de Gemini se consume por HTTP directo desde el service worker.

**Sistemas externos**
- `generativelanguage.googleapis.com` — `POST /v1beta/interactions` con header `x-goog-api-key`, sin SDK de cliente.
- YouTube (`youtube.com/watch`) — se observa la red del reproductor; no se modifica el vídeo ni se altera la cuenta del usuario.

**Coste esperado**: ~$0,006 por cada 10 min de vídeo traducido (~$0,03/hora) con el modelo `flash-lite`. En free tier es $0, con la salvedad de que Google usa el contenido para mejorar sus productos.

**Riesgo principal**: el mecanismo de carga de captions de YouTube puede cambiar sin aviso. Mitigación: la cascada de 4 rutas, el hook sobre `fetch` **y** `XHR`, y el verificador de completitud, que escala de ruta antes de aceptar un resultado truncado.

**Fuera de alcance (cambios posteriores previstos)**
- `add-browser-tts-engine` — motor de locución `speechSynthesis` con mitigaciones del bug de corte a los 15 s de Chromium.
- `add-gemini-tts-engine` — motor Gemini TTS, retimificación exacta y modelo de audio "tipo dub" (original a 0 % durante el diálogo, música original intacta en los huecos).
- `polish-extension-ux` — atajos de teclado, casos degenerados, panel de depuración, soporte de Firefox.
