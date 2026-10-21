# Plan de Desarrollo: Extensión de Doblaje de YouTube con WXT e IA (Gemini)

> **Estado:** revisado · **Fecha:** 2026-10-21 · **Stack:** WXT 0.21.4 + SolidJS + Gemini 3.5 Flash-Lite

Extensión de navegador que extrae la pista de subtítulos **completa** de un vídeo de YouTube, la adapta al español con IA respetando un **presupuesto de tiempo por línea**, y la reproduce sincronizada mediante TTS.

---

## 0. Resumen ejecutivo

El objetivo es "doblar" un vídeo de YouTube al español sin que la traducción desborde los tiempos originales. El problema real no es traducir: es que **el español necesita ~1,25× las palabras del inglés** para el mismo contenido. Si no se comprime con criterio, la locución se solapa.

Por tanto la arquitectura tiene **tres pilares**:

1. **Extracción por intercepción, no por fetch.** El reproductor de YouTube ya pide los subtítulos con un *proof-of-origin token* (PO token) válido. Interceptamos esa petición desde el *main world* y además **parcheamos la URL en vuelo** para forzar `fmt=json3`. Así obtenemos la pista **entera, del vídeo completo, en una sola petición** y con atestación válida. No hay ningún token que gestionar.
2. **Presupuesto de palabras en el prompt.** Cada cue recibe un límite duro de palabras derivado de su duración y de la tasa de habla del motor TTS. El modelo rellena ese presupuesto; si no cabe, recorta relleno; puede fusionar cues contiguos para ganar espacio. Se valida en cliente y se reintenta si se pasa.
3. **Entrega progresiva por bloques.** El guion se traduce en ventanas de 2 min con *look-ahead* de 45 s, así el doblaje arranca ~20 s después de abrir el vídeo, no al final.

**Motor TTS dual:** `speechSynthesis` del navegador (gratis, instantáneo, offline) por defecto, y **Gemini TTS** (`gemini-3.8-flash-lite-tts`) como modo premium. El segundo es superior porque produce audio real, lo que permite **re-timificar** el doblaje (conocer la duración exacta y recalcular los tiempos) y funciona en segundo plano — el TTS del navegador no.

### Veredicto de viabilidad

| Fase | Riesgo | Viabilidad |
|---|---|---|
| Extracción de subtítulos | **Alto** — rompe si se hace con `fetch` directo | ✅ Resuelto con intercepción MAIN world + patch de URL |
| Traducción/adaptación | Bajo | ✅ `gemini-3.5-flash-lite` + structured output |
| TTS navegador | **Medio** — bug de 15 s de Chromium, throttling en background | ⚠️ Viable con mitigaciones; ver §5.3.1 |
| TTS Gemini | Bajo | ✅ Resuelve los dos problemas del TTS navegador |
| Sincronización | Medio | ✅ Puntero + `requestVideoFrameCallback` |

**El único punto crítico es la Fase 1 (extracción).** Todo lo demás es ingeniería ordinaria.

---

## 1. Correcciones al plan original

El plan original (v1) tenía cinco problemas que lo impedían funcionar tal cual. Se listan aquí porque explican la mayoría de las decisiones de diseño que siguen.

### 1.1 🔴 `fetch` directo a `timedtext` devuelve cuerpo vacío

Desde ~2025, el cliente web de YouTube exige un PO token (proof-of-origin) para las URLs de `/api/timedtext`. Una petición directa —incluso con las cookies del usuario— responde **HTTP 200 con el cuerpo vacío**. No es un error: es un éxito sin datos. Está documentado en el ADR de `youtube-transcript-to-pdf`, en `yt-youtube-transcript` y en la guía de PO Tokens de yt-dlp.

**Corrección:** no descargamos los subtítulos nosotros; **interceptamos al reproductor** mientras los descarga. Esa petición ya viene firmada.

### 1.2 🔴 `ytplayer.config` está obsoleto y el content script no lo ve

`ytplayer.config` es de la era pre-2018. Hoy el equivalente es `ytInitialPlayerResponse`. Además, los content scripts corren en el **isolated world**: comparten el DOM pero no las variables JS de la página, así que `window.ytInitialPlayerResponse` es `undefined` para nosotros.

**Corrección:** todo lo que necesite el player response (lista de pistas, `baseUrl`, duración) se lee desde un script inyectado en el **main world** vía `injectScript` de WXT.

### 1.3 🔴 Manifiesto con sintaxis inválida y permisos incompletos

`*://*://*` no es un match pattern válido. Y faltaban los `host_permissions` para la API de Gemini y para que el content script pueda hacer `fetch` a YouTube sin CORS.

**Corrección:** manifiesto en §7.

### 1.4 🟡 `gemini-2.5-flash` está restringido

La documentación oficial indica que el acceso a los modelos 2.5 se limita a usuarios que los hayan usado anteriormente, y recomienda `3.5 Flash-Lite` o `3.8 Flash` para proyectos nuevos.

**Corrección:** `gemini-3.5-flash-lite` ($0.30 in / $2.50 out por 1M tokens) — su descripción oficial es *"optimized for high-volume agentic tasks, translation, and simple data processing"*.

### 1.5 🟡 `speechSynthesis` tiene dos bugs que matan la experiencia

- **Corte a los ~15 s** — [crbug 41294170](https://issues.chromium.org/issues/41294170) / [41491635](https://issues.chromium.org/issues/41491635), abierto. Afecta a las voces de Google. *Workaround:* utterances de < 15 s + keepalive `pause()/resume()`.
- **Throttling en background tab** — Chrome/Safari ralentizan o detienen la síntesis cuando la pestaña pierde el foco. El caso de uso típico (YouTube en segundo plano) rompe el doblaje.

**Corrección:** mitigaciones en §5.3.1 y motor Gemini TTS como alternativa real (§5.3.2).

---

## 2. Arquitectura del sistema (WXT 0.21.4)

```
chimpdub/
├── wxt.config.ts                  Manifiesto + permisos + recursos web-accessible
├── entrypoints/
│   ├── background.ts              Service worker: proxy a Gemini, caché, validación
│   ├── youtube.content.ts         Isolated world: coordina todo, UI, reloj, TTS
│   ├── yt-main-world.ts           Unlisted script: hooks de red + acceso al player
│   └── popup/                     SolidJS: API key, motor TTS, compresión, estado
├── composables/
│   ├── captions/
│   │   ├── extract.ts             Orquestación de las 4 rutas + escalada
│   │   ├── parse-json3.ts         Parser JSON3 (formato principal)
│   │   ├── parse-xml.ts           Parser srv3 / srv1
│   │   ├── clean.ts               Limpieza de entidades, tags, timestamps inline
│   │   └── verify.ts              Verificador de completitud
│   ├── ai/
│   │   ├── translate.ts           Prompt + schema + validación + reintento
│   │   ├── prompt.ts              System instruction + builder de cues
│   │   └── tts-gemini.ts          Pre-generación de audio WAV + retimificación
│   ├── tts/
│   │   ├── browser.ts             speechSynthesis + mitigaciones Chromium
│   │   └── types.ts               Interfaz común de motor TTS
│   ├── player/
│   │   ├── controller.ts          setVolume/getVolume, setOption, loadModule
│   │   └── clock.ts               requestVideoFrameCallback + puntero de cues
│   └── budget.ts                  Cálculo de presupuesto de palabras (WPS)
├── utils/
│   ├── types.ts                   Cue, DubLine, Track, Config
│   └── storage.ts                 Items tipados de storage.defineItem
└── docs/plan.md
```

### Responsabilidades por entrypoint

| Entrypoint | Mundo | Responsabilidad |
|---|---|---|
| `background.ts` | Service worker | Proxy seguro a Gemini (la API key nunca sale de aquí), caché por `videoId`, reintentos |
| `youtube.content.ts` | Isolated world | Reloj de reproducción, cola de TTS, ducking de volumen, UI in-page, orquestación |
| `yt-main-world.ts` | Main world | Hook de `fetch`/`XHR` sobre `/api/timedtext`, patch de URL, lectura del player response, control de volumen del player |
| `popup/` | Extension page | API key, selección de motor TTS y voz, nivel de compresión, estado de la traducción |

**Comunicación main world ↔ isolated world:** `CustomEvent` sobre `window` (`chimpdub:captions`, `chimpdub:player`), ya que `injectScript` devuelve el elemento `<script>` inyectado y permite `dispatchEvent` en ambos sentidos.

---

## 3. Flujo de datos

```
[Abrir vídeo]
      │
      ▼
[content script · run_at: document_start]
      │  injectScript('/yt-main-world.js')
      ▼
[MAIN WORLD] ── hook fetch/XHR sobre /api/timedtext
      │         ── patch URL: &fmt=json3  (&tlang=xx opcional)
      │
      │  ¿llegó el body?
      ├──── NO ──► [Ruta 1] getPlayerResponse() → baseUrl → fetch directo
      │                  └─ cuerpo vacío? ──► [Ruta 2] panel de transcripción DOM
      │                                        └─ [Ruta 3] scrape del HTML del watch page
      ▼
[SÍ] parse json3 ──► clean() ──► verify() contra lengthSeconds
      │                                    │
      │                            ¿completo? ── NO ──► escala a Ruta 2
      ▼
[cache: local:captions:<videoId>]
      │
      ▼
[content script] ──► background: bloques de 120s con look-ahead 45s
      │                        │
      │                        ▼
      │              [background] ──► Gemini 3.5 Flash-Lite
      │                                 generationConfig.responseSchema
      │                                 (structured output: no puede romper formato)
      │                        │
      │                        ▼
      │              [validar: words ≤ budget, start<end, monótono, sin solapes]
      │                        │  ¿falla? ──► reintento (1x) ──► fallback texto original
      │                        ▼
      │              [session:dub:<videoId>:<block>]
      ▼
[content script] ◄─── bloque listo (progresivo)
      │
      ▼
[RELOJO] requestVideoFrameCallback(100ms) + puntero de cues
      │
      ├── t >= cue.start y no hablado ──► hablar
      │        ├── motor browser: speechSynthesis.speak()  (utterance < 15s + keepalive)
      │        └── motor gemini:  new Audio(blobUrl)         (pre-generado, duración conocida)
      │
      ├── al empezar ──► movie_player.setVolume(0.15)   (ducking)
      └── al terminar ──► movie_player.setVolume(original)  (restore)
```

---

## 4. Fases de desarrollo

Cada fase tiene un **criterio de aceptación** verificable. No se empieza la siguiente hasta que la anterior se cumple.

### Fase 0 — Andamiaje y configuración

**Qué:**
- `wxt.config.ts` con el manifiesto real de §7 (permisos, `host_permissions`, `web_accessible_resources`).
- Renombrar `entrypoints/content.ts` → `entrypoints/youtube.content.ts` con `matches: ['*://*.youtube.com/*']` y `run_at: 'document_start'`.
- Popup Solid: campo de API key (persistida), selector de motor TTS, selector de voz, nivel de compresión, toggle on/off, estado.
- `utils/storage.ts` con items tipados (`storage.defineItem`).

**Aceptación:** la extensión carga en Chrome, el popup guarda y recupera la API key, y el content script aparece en `chrome://extensions` → "Inspect views".

---

### Fase 1 — Extracción de subtítulos *(fase crítica)*

**Qué:**
- `yt-main-world.ts`: hook de `fetch` **y** `XMLHttpRequest` sobre `/api/timedtext`, con patch de URL en vuelo.
- Forzado de la petición del player: `unloadModule('captions')` + `setOption('captions','track',{languageCode})`.
- Enrutado por `vssId` y soporte de `translationLanguage` para obtener el idioma fuente.
- Parsers: `json3` (principal), `srv3`/`srv1` (XML), fallback a `vtt`.
- `clean()`: entidades HTML, tags `<c>`/`<i>`/`<b>`, timestamps inline de VTT, saltos de línea, `[Music]`/`(aplausos)`.
- `verify()`: cobertura, hueco máximo, cola contra `lengthSeconds`.
- Cascada de 4 rutas con escalada automática (§5.1.6).
- Caché de cues crudas por `videoId`.

**Aceptación:** sobre un vídeo de ≥ 10 min con subtítulos, la extensión extrae **la pista completa** (último cue ≈ duración del vídeo, sin huecos > 30 s) y la lista de pistas disponibles, incluyendo ASR vs manual. Funciona con sesión anónima y con sesión iniciada.

---

### Fase 2 — Traducción y adaptación

**Qué:**
- `background.ts`: `POST https://generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash-lite:generateContent` con header `x-goog-api-key`.
- `generationConfig.responseMimeType: 'application/json'` + `responseSchema` (§5.2.3) — **structured output**, el modelo no puede devolver formato inválido.
- System instruction con presupuesto de palabras (§5.2.1), permitir fusión/división de cues.
- Chunking progresivo: ventanas de 120 s con look-ahead de 45 s y contexto de cola del bloque anterior.
- Validación en cliente + 1 reintento con prompt más estricto + fallback al texto original.
- Caché del guion adaptado por `videoId` + `promptVersion`.

**Aceptación:** un bloque de 2 min se adapta respetando `words ≤ budgetWords` en el 100% de los casos (tras reintento), y el primer bloque está disponible < 15 s después de abrir el vídeo.

---

### Fase 3 — Motor de sincronización y subtítulos traducidos

> Este es el MVP: valor real, coste ~$0.007 por 10 min de vídeo, y no depende de los bugs de TTS.

**Qué:**
- `player/clock.ts`: `video.requestVideoFrameCallback()` (Chrome/Edge/Safari 15.4+) con fallback a `setInterval(100 ms)`. **No** `timeupdate` (dispara a ~4 Hz → 250 ms de error).
- Puntero de cues: nunca igualdad de timestamps (§5.4.1).
- Detección de `seeking` / `pause` → reset del puntero + cancelación de voz.
- Renderizado de los subtítulos traducidos sobre el reproductor (contenedor propio, `createShadowRootUi` para estilos aislados).
- Badge de estado en la página: "extrayendo…", "traduciendo bloque N…", "listo".

**Aceptación:** los subtítulos en español se muestran sincronizados con error < 150 ms respecto al cue original, y sobreviven a pausa, seek y cambio de vídeo (SPA) sin desincronizarse.

---

### Fase 4 — Motor TTS navegador

**Qué:**
- `tts/browser.ts`: cola de utterances, `lang: 'es-ES'`, `rate` configurable (1.0–1.3), selección de voz.
- Mitigaciones de Chromium (§5.3.1): utterances < 15 s, keepalive `pause()/resume()` cada 10 s, watchdog de silencio, `cancel()` en `pause`/`seeking`.
- Ducking de volumen vía `movie_player.setVolume()` (§5.4.2).

**Aceptación:** un vídeo de 3 min se dubla completo sin cortes silenciosos, el volumen original baja a 15% durante la locución y se restaura exactamente al valor previo del usuario, y no hay solapamiento entre locuciones.

---

### Fase 5 — Motor TTS Gemini (modo premium)

**Qué:**
- Pre-generación de audio por bloque: `gemini-3.8-flash-lite-tts` vía `POST /v1beta/interactions` con `response_format: {type:'audio', mime_type:'audio/wav'}` → base64 → `Blob` → `URL.createObjectURL` → `new Audio()`.
- **Buffer de 1 bloque**ahead: mientras se reproduce el bloque N, se genera el N+1.
- **Retimificación**: al conocer la duración real del audio, recalcular `start`/`end` de cada línea para que encaje y no solape (§5.4.3).
- Recuperación deAudio: si el TTS no está listo cuando toca la línea, se usa el navegador como fallback.

**Aceptación:** en modo Gemini el doblaje suena con voz de estudio, el sync es exacto al sample, **funciona con la pestaña en segundo plano**, y el reintento/buffer no produce silencios en la transición de bloques.

---

### Fase 6 — UX, robustez y pulido

**Qué:**
- `wxt:locationchange` para reaccionar al cambio de vídeo sin recargar (YouTube es SPA).
- Detección de casos degenerados: sin subtítulos, solo ASR, vídeo ya en español, vídeo restringido.
- Aviso al usuario si la única pista es ASR (la calidad del doblaje depende de ello).
- Panel de depuración: cues originales vs adaptados vs locución, con marca de presupuesto excedida.
- Atajos de teclado: activar/desactivar doblaje, saltar cue, repetir cue.
- Build para Firefox (`wxt build -b firefox`): `injectScript` funciona, `world: 'MAIN'` nativo no.

**Aceptación:** cambiar de vídeo dentro de YouTube re-inicializa correctamente el pipeline sin recargar la pestaña, y los casos degenerados muestran un mensaje claro en lugar de fallar en silencio.

---

### Fase 7 — (Opcional) Transcripción de audio

Para vídeos **sin** subtítulos ni ASR. Requiere extraer el audio del player (hoy segmentado con SABR) y usar `gemini-3.5-transcribe` ($0.003/min). Proyecto aparte, no bloquea el resto.

---

## 5. Especificaciones detalladas

### 5.1 Extracción de subtítulos

#### 5.1.1 El hook principal (main world)

```ts
// entrypoints/yt-main-world.ts
export default defineUnlistedScript(() => {
  const EMIT = 'chimpdub:captions';

  // --- fetch ---
  const origFetch = window.fetch;
  window.fetch = async (input: any, init?: any) => {
    const url: string | undefined =
      typeof input === 'string' ? input : input?.url;

    if (!url?.includes('/api/timedtext')) return origFetch(input, init);

    // PATCH EN VUELO: pedimos el formato más rico con la atestación del player
    const u = new URL(url);
    u.searchParams.set('fmt', 'json3');

    const newUrl = u.toString();
    const res = await origFetch(
      typeof input === 'string' ? newUrl : new Request(newUrl, input),
      init,
    );

    res.clone()
      .text()
      .then((body) => {
        window.dispatchEvent(new CustomEvent(EMIT, {
          detail: {
            url: newUrl,
            body,
            lang: u.searchParams.get('lang'),
            tlang: u.searchParams.get('tlang'),
            vssId: u.searchParams.get('vss_id'),
          },
        }));
      })
      .catch(() => {});

    return res;
  };

  // --- XHR (por si el player usa XHR en lugar de fetch) ---
  const origOpen = XMLHttpRequest.prototype.open;
  const origSend = XMLHttpRequest.prototype.send;

  XMLHttpRequest.prototype.open = function (method: string, url: string, ...rest: any[]) {
    (this as any).__chimpdubUrl = url;
    return origOpen.call(this, method, url, ...rest);
  };

  XMLHttpRequest.prototype.send = function (...args: any[]) {
    const url = (this as any).__chimpdubUrl;
    if (url?.includes('/api/timedtext')) {
      this.addEventListener('load', () => {
        try {
          const u = new URL(url, location.href);
          u.searchParams.set('fmt', 'json3');
          window.dispatchEvent(new CustomEvent(EMIT, {
            detail: { url: u.toString(), body: this.responseText, lang: u.searchParams.get('lang') },
          }));
        } catch {}
      });
    }
    return origSend.apply(this, args as any);
  };
});
```

#### 5.1.2 Forzar la petición del player

```js
// main world
const p = document.getElementById('movie_player');
p.unloadModule('captions');
p.setOption('captions', 'track', { languageCode: 'en' });
```

**El idioma debe ser explícito.** Si el navegador está en español, el player pediría `es` y no `en`.

Para obtener el idioma **fuente** de un vídeo que solo tiene otra pista:

```js
p.setOption('captions', 'track', {
  languageCode: 'es',              // la pista que existe
  translationLanguage: 'en',       // pedirla traducida al original
});
```

#### 5.1.3 Taxonomía de `vssId` y selección de pista

| `vssId` | Significado | Preferencia |
|---|---|---|
| `en` | Manual en inglés | ⭐ Máxima |
| `a.en` | ASR (auto-generado) en inglés | Media |
| `a.en-orig` | ASR del audio original | Media |
| `.es` | Traducción automática de YouTube | Baja (literal) |
| `a.es` | ASR en español | Variable |

Regla: **manual > ASR** y **idioma del audio original > traducción**. En la práctica es mejor **interceptar todas** las peticiones que lleguen y quedarse con la mejor, en lugar de forzar una sola.

#### 5.1.4 Formatos de respuesta

| `fmt` | Formato | Estructura |
|---|---|---|
| `json3` | JSON | `events[].segs[].utf8`, `tStartMs`, `dDurationMs` — **el que usamos** |
| `srv3` | XML | `<text start="1.23" dur="4.5">` — el que pide el player por defecto |
| `srv1` | XML simple | `<text t="..." d="...">` |
| `vtt` | WebVTT | con tags de estilo y timestamps inline |

> **Corrección respecto al plan v1:** el endpoint `timedtext` **no devuelve SRT**. Hay que pedir `fmt` explícitamente.

#### 5.1.5 Limpieza obligatoria

Antes de enviar nada a Gemini:

- Entidades HTML: `&amp;` `&#39;` `&quot;` `&nbsp;` → carácter
- Tags de formato: `<c.colorE5E5E5>`, `<i>`, `<b>`, `<font>` → eliminar
- Timestamps inline de VTT: `<00:00:01.000>` → eliminar
- Saltos de línea hard-coded y espacios múltiples → normalizar
- Anotaciones de ASR: `[Music]`, `(aplausos)`, `>>` → según política (recomendado: eliminar `[Music]`, conservar el resto como contexto)

Sin esta limpieza, el modelo devuelve basura en el JSON.

#### 5.1.6 Cascada de rutas con escalada

| # | Ruta | ¿Completo? | ¿Atestación válida? | Notas |
|---|---|---|---|---|
| **0** | Hook `fetch`+`XHR` + patch `fmt=json3` + forzar `setOption` | ✅ Sí | ✅ Sí (es la del player) | Ruta principal. Escala a vídeos largos. |
| **0.5** | Ídem + `translationLanguage` para el idioma fuente | ✅ Sí | ✅ Sí | Cuando el vídeo no tiene pista en el idioma fuente |
| **1** | `getPlayerResponse()` → `baseUrl` → `fetch` directo | ⚠️ **Cuerpo vacío** | ❌ | Sirve para obtener la *lista* de pistas |
| **2** | Panel de transcripción del DOM | ✅ Sí (requiere scroll) | ✅ Sí (lo carga la página) | Red de seguridad. **Lento para vídeos largos**: la lista está virtualizada |
| **3** | Scrape del HTML del watch page → `ytInitialPlayerResponse` | Solo la *lista* de pistas | ❌ | Último recurso |

**La ruta 0 debe ser la primaria, no el plan B.** La ruta 2 no escala a vídeos de más de ~20 min por la virtualización de la lista.

#### 5.1.7 Verificación de completitud

```ts
export function verifyCompleteness(cues: Cue[], durationSec: number) {
  const dur = durationSec || 0;
  const lastEnd = cues.at(-1)?.end ?? 0;
  const coverage = cues.reduce((a, c) => a + (c.end - c.start), 0) / (dur || 1);

  let maxGap = 0;
  for (let i = 1; i < cues.length; i++) {
    maxGap = Math.max(maxGap, cues[i].start - cues[i - 1].end);
  }

  const suspicious =
    cues.length < 5 ||        // claramente truncado
    lastEnd < dur * 0.75 ||   // no llega al final del vídeo
    coverage < 0.40 ||        // apenas cubre el vídeo
    maxGap > 30;              // salto enorme en medio

  return { complete: !suspicious, coverage, lastEnd, maxGap, cueCount: cues.length };
}
```

Si `complete === false` → **escalar a la ruta 2**, nunca continuar en silencio con un resultado truncado.

> Nota: un ASR puede tener huecos inherentes (silencios, música). Eso no es un fallo de extracción; por eso los umbrales son heurísticos y no estrictos.

---

### 5.2 Traducción y adaptación

#### 5.2.1 El presupuesto de palabras (el corazón del proyecto)

El español requiere ~1,25× las palabras del inglés. El TTS habla ~2,9 palabras/s a `rate 1.1`. Por tanto, para un cue de `dur` segundos:

```
budgetWords = floor(dur × WPS_TARGET × SAFETY)
```

| Parámetro | Valor | Nota |
|---|---|---|
| `WPS_TARGET` | 2,9 | palabras/s a `rate 1.1` — **calibrar empíricamente** con la voz elegida |
| `SAFETY` | 0,92 | margen para latencia de arranque y variabilidad |

Ejemplo: cue de 3,0 s → `floor(3,0 × 2,9 × 0,92)` = **8 palabras**. El inglés original de 3 s son ~8 palabras → en español harían ~10 → hay que cortar ~2. Ese es el presupuesto real.

El prompt **no** dice "recorta drásticamente" (criterio cero, resultado arbitrario). Dice cuánto cabe.

#### 5.2.2 System instruction

```
Eres un adaptador de subtítulos para doblaje en español.

REGLAS
1. Traduce a español natural, no literal. Suena a doblaje, no a traducción.
2. Cada línea tiene un presupuesto de palabras. NO PUEDES superarlo.
   Si no cabe, recorta en este orden:
     a) muletillas y relleno ("o sea", "bueno", "vale", "en plan", "like")
     b) redundancias y repeticiones
     c) adverbios y adjetivos prescindibles
   CONSERVA siempre: nombres propios, cifras, fechas, datos concretos,
   negaciones, y el sentido general de la frase.
3. Puedes FUSIONAR dos cues contiguos si el presupuesto combinado da más
   espacio. Devuelve el start del primero y el end del segundo, y lista
   ambos índices en "source".
4. Puedes DIVIDIR un cue demasiado largo en dos líneas, repartiendo el
   presupuesto. Ambas líneas deben caber.
5. No inventes información. No añadas explicaciones ni acotaciones.
6. El texto se LEE en voz alta: sin asteriscos, sin corchetes, sin etiquetas,
   sin números de cue, sin comillas decorativas.
7. Los tiempos que devuelves pueden ser distintos de los originales, pero
   deben ser monotónicos crecientes y no solaparse.
```

#### 5.2.3 Structured output (structured outputs)

```json
{
  "generationConfig": {
    "responseMimeType": "application/json",
    "responseSchema": {
      "type": "ARRAY",
      "items": {
        "type": "OBJECT",
        "properties": {
          "start":  { "type": "NUMBER" },
          "end":    { "type": "NUMBER" },
          "text":   { "type": "STRING" },
          "words":  { "type": "INTEGER" },
          "source": { "type": "ARRAY", "items": { "type": "INTEGER" } }
        },
        "required": ["start", "end", "text", "words", "source"]
      }
    }
  }
}
```

Con `responseSchema`, el modelo **no puede** devolver un formato que rompa el parser. Esto elimina de raíz el requisito del plan v1 de "asegurar que devuelva la misma estructura sin romper el formato".

`source` guarda los índices de los cues originales fusionados: invaluable para depuración y para el fallback de subtítulos.

#### 5.2.4 Chunking progresivo

| Parámetro | Valor | Razón |
|---|---|---|
| Tamaño de ventana | 120 s | Bloque suficientemente grande para contexto, suficientemente pequeño para reintento barato |
| Look-ahead | 45 s | El bloque N se pide cuando el playback entra en su ventana |
| Contexto de cola | ~10 s del bloque anterior | Evita que el modelo corte una frase a la mitad |

Con look-ahead de 45 s, el primer bloque está listo ~20 s después de abrir el vídeo, no al final.

**Enviar el vídeo entero de una vez** da igual (1M de contexto), pero se trocea igualmente por dos razones prácticas:

#### 5.2.5 Validación en cliente

```ts
function validate(dub: DubLine[], cues: Cue[]): 'ok' | 'over-budget' | 'invalid' {
  let prevEnd = -1;
  for (const d of dub) {
    if (!(d.end > d.start)) return 'invalid';
    if (d.start < prevEnd) return 'invalid';       // solape o desorden
    if (d.words > budgetWords(cues, d)) return 'over-budget';
    if (!d.text.trim()) return 'invalid';
    prevEnd = d.end;
  }
  return 'ok';
}
```

- `over-budget` → **1 reintento** con prompt más estricto.
- `invalid` → reintento; si persiste, **fallback al texto original** de ese cue (el doblaje continúa, aunque sea en inglés o con la traducción literal).

#### 5.2.6 Caché

| Dato | Clave | Área |
|---|---|---|
| Cues crudas | `local:captions:<videoId>:<lang>` | `storage.local` |
| Guion adaptado | `local:dub:<videoId>:<lang>:<promptVersion>` | `storage.local` |
| Bloque en curso | `session:dub:<videoId>:<block>` | `storage.session` |
| API key | `local:apiKey` | `storage.local` |

`promptVersion` permite invalidar la caché cuando cambia el prompt. El `baseUrl` de YouTube **expira** → los cues crudos sí se cachean, el `baseUrl` no.

---

### 5.3 Motores de TTS

#### 5.3.1 Motor navegador (`speechSynthesis`)

```ts
const MAX_UTTERANCE_MS = 12_000;  // < 15 s (límite de Chromium)
const KEEPALIVE_MS = 10_000;       // pause()/resume() para evitar el corte

export class BrowserTts implements TtsEngine {
  private synth = speechSynthesis;
  private keepalive?: number;

  speak(text: string, opts: { rate: number; voiceURI?: string; onEnd: () => void }) {
    // 1) Trocear por frases si supera el límite
    for (const chunk of splitBySentences(text, MAX_UTTERANCE_MS)) {
      const u = new SpeechSynthesisUtterance(chunk);
      u.lang = 'es-ES';
      u.rate = opts.rate;
      if (opts.voiceURI) u.voice = pickVoice(opts.voiceURI);
      u.onend = opts.onEnd;
      this.synth.speak(u);
    }

    // 2) Keepalive contra crbug 41294170
    this.keepalive = window.setInterval(() => {
      if (this.synth.speaking && !this.synth.paused) {
        this.synth.pause();
        this.synth.resume();
      }
    }, KEEPALIVE_MS);
  }

  cancel() {
    this.synth.cancel();
    if (this.keepalive) clearInterval(this.keepalive);
  }
}
```

**Mitigaciones obligatorias:**

| Problema | Mitigación |
|---|---|
| Corte a los 15 s (crbug 41294170) | Trocear utterances a < 12 s + keepalive `pause()/resume()` |
| Throttling en background tab | **No tiene arreglo** → es el motivo principal del motor Gemini (§5.3.2) |
| `onboundary` no fiable en Chrome (~10% de voces) | No depender de resaltado palabra a palabra |
| Voces cargadas de forma asíncrona | `voiceschanged` + `getVoices()` con reintento |
| Cola de utterances pendientes | Una locución activa como máximo; `cancel()` de la anterior si llega la siguiente |

#### 5.3.2 Motor Gemini (premium)

```ts
// POST https://generativelanguage.googleapis.com/v1beta/interactions
{
  "model": "gemini-3.8-flash-lite-tts",
  "input": [{
    "type": "user_input",
    "content": [{
      "type": "text",
      "text": "Buenos días. <short pause> ¿En qué podemos ayudar?",
      "annotations": [{ "type": "speech_metadata", "style": "neutral, natural pace" }]
    }]
  }],
  "response_format": { "type": "audio", "mime_type": "audio/wav" },
  "generation_config": { "speech_config": [{ "voice": "Kore" }] }
}
```

**Ventajas que el navegador no tiene:**

1. **Re-timificación.** Al ser audio real, se conoce la duración exacta → se pueden recalcular los `start`/`end` de todas las líneas para que encajen y no solapen (§5.4.3). Con TTS de navegador la duración es impredecible y el ajuste es a ciegas.
2. **Funciona en segundo plano** (sin throttling).
3. **Voz de estudio**, 130+ idiomas, control de estilo y énfasis, 30 voces predefinidas + biblioteca extendida.
4. **Precisión de sample** con `Audio` + `currentTime`.

**Inconvenientes y mitigaciones:**

| Inconveniente | Mitigación |
|---|---|
| Latencia ~1-2 s por llamada | Buffer de 1 bloque ahead |
| Coste | $0.0015/10 s (flash-lite) → ~$0.09 por 10 min de vídeo. Free tier disponible |
| Descarga de blobs (base64 WAV 24 kHz mono) | `Blob` + `URL.createObjectURL`; revocar al terminar |
| Cuota de `storage.local` (5 MB) | **No cachear audio en storage.** Regenerar o mantener solo en memoria |
| No es offline | Modo navegador como fallback |

---

### 5.4 Sincronización y audio

#### 5.4.1 Puntero de cues (nunca igualdad de timestamps)

`currentTime == cue.start` **nunca se cumple** (flotantes), y `timeupdate` dispara a ~4 Hz.

```ts
class DubClock {
  private idx = 0;
  private spoken = new Set<number>();
  private lastT = 0;

  tick(t: number) {
    // Detección de seek: salto grande en el reloj
    if (Math.abs(t - this.lastT) > 1.5) {
      this.reset(t);
      return;
    }
    this.lastT = t;

    // Avanzar el puntero (monótono, nunca atrás)
    while (this.idx < this.lines.length && this.lines[this.idx].end <= t) {
      this.idx++;
    }

    const line = this.lines[this.idx];
    if (line && t >= line.start && !this.spoken.has(this.idx)) {
      this.spoken.add(this.idx);
      this.onLine?.(line, this.idx);
    }
  }

  reset(t: number) {
    this.idx = 0;
    this.spoken.clear();
    this.lastT = t;
    while (this.idx < this.lines.length && this.lines[this.idx].end <= t) this.idx++;
  }
}
```

Reloj: `video.requestVideoFrameCallback()` con fallback a `setInterval(100 ms)`. La latencia de arranque del TTS (~100-300 ms) se compensa disparando ~150 ms antes del `start` del cue.

#### 5.4.2 Ducking de volumen

`video.volume` **no es fiable** en YouTube (el player gestiona su propio volumen y lo sobrescribe). Hay que usar la API del player, desde main world:

```js
const p = document.getElementById('movie_player');
const original = p.getVolume();   // guardar SIEMPRE antes de tocar nada
p.setVolume(0.15);                 // atenuar  → NO usar mute() (muestra toast)
p.setVolume(original);             // restaurar
```

Usar `mute()`/`unmute()` dispara el aviso "Vídeo silenciado" de YouTube. `setVolume()` no.

Estado: `original` se guarda al iniciar la sesión y se restaura al pausar, hacer seek, desactivar la extensión o descargar la página.

#### 5.4.3 Retimificación (solo con motor Gemini)

```
1. Generar el audio de cada línea → medir su duración real (segundos)
2. Recorrer las líneas en orden:
     cursor = max(cue.start, cursor + GAP_MIN)
     nueva.start = cursor
     nueva.end   = cursor + durReal
     cursor      = nueva.end + GAP_MIN
3. Si una línea se solapa con la siguiente, se recorta o se salta
   (según el presupuesto de la línea original)
```

`GAP_MIN` ≈ 0,15 s. Esto absorbe la latencia de arranque y garantiza que la locución termine antes de que empiece la siguiente.

Con motor navegador, la duración es **desconocida** → se usa la estimación `words / WPS_TARGET` y se aplica el mismo algoritmo, pero el resultado es aproximado.

#### 5.4.4 Cancelación de emergencia

| Evento | Acción |
|---|---|
| `pause` | `tts.cancel()` + restaurar volumen + reset del puntero |
| `seeking` | `tts.cancel()` + restaurar volumen + reset del puntero |
| Llega la línea N+1 y N sigue hablando | `tts.cancel()` de N (regla: **una locución activa como máximo**) |
| Cambio de vídeo (`wxt:locationchange`) | Cancelar todo + limpiar estado |
| Desactivar la extensión | Cancelar todo + restaurar volumen |

---

## 6. Autenticación, claves y login

Hay **tres tokens distintos** en el sistema y conviene no confundirlos:

### 6.1 API key de Gemini — sí requiere login (una vez)

- Se obtiene en **https://aistudio.google.com/apikey** con una cuenta de Google. Si eres nuevo, AI Studio crea automáticamente un proyecto de GCP por defecto y una key al aceptar los ToS.
- Es una **cadena estática** (`AIza...`), no un OAuth. No hay refresh, ni "Sign in with Google" dentro de la extensión, ni backend proxy.
- **Desde el 28 de mayo de 2026**, toda key nueva creada en AI Studio es una **auth key**: ligada a una service account, restringida a la Generative Language API, con *fast-acting leaked key enforcement*. Además, la API **rechaza las standard keys sin restricciones**. O sea: la key que se cree hoy ya viene restringida de fábrica.
- **Restricciones por origen no son aplicables** a una extensión (el origen `chrome-extension://<id>` cambia según la instalación, y en Firefox el UUID es aleatorio). La restricción "solo Gemini API" sí sirve y ya viene por defecto.
- Patrón **BYO key**: el usuario pega su key, se guarda en `storage.local`, nunca se hardcodea en el bundle, y la página de YouTube nunca la ve. Solo la usa el service worker, que hace la llamada con `host_permissions` (en MV3 esto salta el CORS → **no hace falta proxy**).
- Si se filtra, alguien consume tu cuota → **billing alerts** en la consola de GCP.

### 6.2 PO token de YouTube — no requiere login ni se gestiona

Lo emite automáticamente el BotGuard de YouTube **dentro de la sesión real del navegador**, y el reproductor ya lo lleva. Por eso:

- ❌ `fetch(captionUrl)` desde el content script → **HTTP 200 con cuerpo vacío**.
- ✅ Interceptar la petición que hace el player → ya viene firmada.

La extensión **hereda** el token, nunca lo toca. De ahí que no haya ninguna credencial que almacenar.

### 6.3 Sesión de YouTube — solo en casos concretos

| Situación | ¿Sesión logueada? |
|---|---|
| Vídeo público normal | No hace falta |
| Age-restricted (18+), privado, solo miembros | **Sí** |
| Recomendación general | **Sí, por comodidad** — la aporta el navegador, no la extensión, y no cuesta nada |

La extensión no pide ni gestiona ese login: simplemente hereda la sesión del navegador a través del main world.

---

## 7. Manifiesto

```ts
// wxt.config.ts
import { defineConfig } from 'wxt';

export default defineConfig({
  modules: ['@wxt-dev/module-solid'],

  manifest: {
    name: 'ChimpDub',
    description: 'Doblaje en español de vídeos de YouTube con IA',
    permissions: ['storage', 'tabs'],

    host_permissions: [
      'https://generativelanguage.googleapis.com/*',   // API de Gemini
      '*://*.youtube.com/*',                            // fetch de captions sin CORS
    ],

    // WXT NO auto-registra los unlisted scripts en web_accessible_resources
    // (solo los CSS de content scripts) → hay que declararlo a mano.
    web_accessible_resources: [
      {
        resources: ['yt-main-world.js'],
        matches: ['*://*.youtube.com/*'],
      },
    ],
  },
});
```

**Notas de manifiesto:**

- `matches: ['*://*://*']` del plan v1 **no es válido**. El content script usa `*://*.youtube.com/*`.
- `tabs` es necesario si el popup lee la URL de la pestaña activa; si solo usa `browser.tabs.query` con `activeTab`, puede bastar `activeTab`.
- `host_permissions` para `*://*.youtube.com/*` evita problemas de CORS cuando el content script hace `fetch` al `baseUrl` de los captions (ruta 1 de la cascada).
- MV2 (Firefox): WXT convierte `host_permissions` → `permissions` y `web_accessible_resources` → array plano automáticamente.

---

## 8. Costes

Precios verificados en la tabla oficial de la Gemini API (Oct 2026).

### 8.1 Traducción

`gemini-3.5-flash-lite` — $0.30 in / $2.50 out por 1M tokens (Standard). **Free tier disponible.**

| Vídeo | Tokens in | Tokens out | Coste (paid) |
|---|---|---|---|
| 10 min | ~2.000 | ~1.800 | **~$0.006** |
| 1 hora | ~12.000 | ~11.000 | **~$0.030** |

### 8.2 TTS

| Modelo | Por 10 s de audio | 10 min de vídeo | 1 hora de vídeo |
|---|---|---|---|
| `gemini-3.8-flash-lite-tts` | $0.0015 | **~$0.09** | ~$0.54 |
| `gemini-3.8-flash-tts` (calidad) | $0.00225 | ~$0.135 | ~$0.81 |
| `speechSynthesis` (navegador) | $0 | **$0** | **$0** |

**Free tier:** ambos modelos TTS son "Free of charge" en free tier. Para uso personal, el coste esperado es **$0**.

⚠️ **Privacidad:** en free tier, la tabla marca `Used to improve our products: **Yes**` — los subtítulos de los vídeos que veu se envían a Google. En plan de pago es `No`. Para contenido sensible, usar plan de pago.

---

## 9. Riesgos y mitigaciones

| # | Riesgo | Impacto | Mitigación |
|---|---|---|---|
| 1 | YouTube cambia el mecanismo de carga de captions | **Alto** | Cascada de 4 rutas + verificación de completitud; el hook intercepta `fetch` **y** `XHR` |
| 2 | YouTube cambia el DOM del player | Medio | El acceso al player está aislado en `player/controller.ts` con fallbacks |
| 3 | Bug de 15 s en `speechSynthesis` | Medio | Utterances < 12 s + keepalive; motor Gemini como salida |
| 4 | Throttling de TTS en background tab | Medio | Idem → motor Gemini |
| 5 | El modelo se pasa del presupuesto de palabras | Medio | Validación + 1 reintento + fallback al texto original |
| 6 | La calidad del ASR en inglés es mala | Medio | Mostrar aviso de "auto-generado" y dejar elegir pista |
| 7 | Formato de subtítulos inesperado | Bajo | Parsers para `json3`, `srv3`, `srv1`, `vtt` |
| 8 | Cuota de `storage.local` (5 MB) | Bajo | No cachear audio; cues por vídeo en `storage.local` |
| 9 | Service worker de MV3 dormido (30 s) | Bajo | `browser.runtime.sendMessage` mantiene vivo el worker durante la promesa; reintento en el content script |
| 10 | Fuga de la API key | Bajo | `storage.local`, BYO key, restricción "solo Gemini API" por defecto |
| 11 | Coste inesperado con TTS | Bajo | Modo navegador por defecto; mostrar coste estimado en el popup |
| 12 | Rutas con breaking changes en la API de Gemini | Bajo | Usar alias `latest` en los modelos y vigilar los avisos de deprecación (2 semanas de margen) |

---

## 10. Decisiones tomadas (y por defecto en este documento)

| Decisión | Valor por defecto | Rationale |
|---|---|---|
| Motor TTS | **Dual**: navegador por defecto + Gemini opcional | El navegador es gratis e instantáneo; Gemini resuelve el background tab y la retimificación |
| Alcance del MVP | **Fase 3** (subtítulos traducidos sincronizados) | Aporta valor real, cuesta ~$0.007/10 min y no depende de los bugs de TTS. La Fase 1 (extracción) es la de más riesgo y conviene validarla primero |
| Estrategia de compresión | **Presupuesto automático** (`maxWords = floor(dur × WPS × 0.92)`) + fusión/división de cues | Convierte "recorta drásticamente" en un criterio verificable |
| Modelo de traducción | `gemini-3.5-flash-lite` | Recomendado oficialmente para traducción de alto volumen |
| Formato de subtítulos | `json3` (parcheado en vuelo) | El más rico en timestamps |
| Chunking | 120 s / look-ahead 45 s | Entrega progresiva y reintentos baratos |
| Idioma de salida | `es-ES` (configurable) | — |
| Rate del TTS navegador | 1.1 (rango 1.0-1.3) | Compromiso entre naturalidad y presupuesto de tiempo |
| Ducking | `setVolume(0.15)` | No usar `mute()` (evita el toast de YouTube) |
| Caché de guion | `local:dub:<videoId>:<lang>:<promptVersion>` | Invalidable al cambiar el prompt |

### Decisiones pendientes de confirmar

1. **Firefox como objetivo de primera clase** — `injectScript` funciona, pero `speechSynthesis` en Firefox tiene menos voces y distinto comportamiento. ¿Firefox es prioridad o solo Chromium?
2. **Modo sin cuenta de YouTube** — ¿se soporta explícitamente el caso "vídeo público, sesión anónima"? (hoy funciona, pero conviene decidir si se documenta o se avisa).
3. **Idiomas de destino** — ¿solo español, o el motor debe ser genérico (`es-ES` / `es-MX` / `pt-BR` / `en-US`)? Afecta al prompt y a la selección de voz.
4. **Múltiples voces / speakers** — para vídeo con diálogo entre varios hablantes, ¿interesa distinguir voces por hablante? (Gemini TTS lo soporta con multi-speaker; `speechSynthesis` no).

---

## 11. Resumen de cambios respecto al plan v1

| Área | Plan v1 | Plan v2 |
|---|---|---|
| Obtención de subtítulos | `fetch` a `ytplayer.config` | Intercepción MAIN world + patch de URL `fmt=json3` + 4 rutas + verificador |
| Formato | SRT/JSON | `json3` / `srv3` / `srv1` / `vtt` (no SRT) |
| Modelo | `gemini-2.5-flash` | `gemini-3.5-flash-lite` + `gemini-3.8-flash-lite-tts` |
| Salida del modelo | "mismos timestamps, que no rompa el formato" | `responseSchema` (structured output) + `start`/`end` propios + `source` |
| Criterio de compresión | "recorta drásticamente" | Presupuesto de palabras por cue + fusión/división + validación |
| Traducción | Por lotes, sin specify ventana | Ventanas de 120 s con look-ahead 45 s y contexto de cola |
| TTS | `speechSynthesis` | Dual: navegador (con mitigaciones) + Gemini TTS |
| Sincronización | `timeupdate` + igualdad de timestamps | `requestVideoFrameCallback` + puntero monótono + retimificación |
| Ducking | `video.volume` | `movie_player.setVolume()` con restore del valor original |
| Manifiesto | `*://*://*` | `*://*.youtube.com/*` + `host_permissions` + `web_accessible_resources` |
| SPA | (no contemplado) | `wxt:locationchange` |
| Caché | "memoria local del hilo activo" | `storage.local` / `storage.session` con items tipados |
| Coste | (no estimado) | ~$0.03/h traducir, ~$0.54/h con TTS Gemini, $0 con free tier |
