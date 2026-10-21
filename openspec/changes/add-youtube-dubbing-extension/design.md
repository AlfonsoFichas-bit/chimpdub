## Context

El repositorio es el starter de WXT + SolidJS: `wxt.config.ts` solo declara el módulo de Solid, `entrypoints/content.ts` es un stub que hace match con `*://*.google.com/*`, `entrypoints/background.ts` es un stub y el popup es el contador de demo. No hay lógica de producto.

El objetivo del producto es doblar vídeos de YouTube al español. La restricción dominante no es la traducción, es **conseguir los subtítulos**: el endpoint `/api/timedtext` exige un PO token emitido por el BotGuard de YouTube dentro de la sesión real del navegador, y una petición propia sin esa atestación responde HTTP 200 con el cuerpo vacío. La consecuencia arquitectónica es que **no se puede "descargar" la pista: hay que observar la descarga que ya hace el reproductor**.

La segunda restricción es de idioma: el español necesita ~1,25× las palabras del inglés. Una traducción literal no cabe en el tiempo del cue original, así que la adaptación necesita un criterio objetivo (presupuesto de palabras) y no una instrucción vaga.

Restricciones de plataforma relevantes:
- Los content scripts corren en **isolated world**: no ven `window.ytInitialPlayerResponse` ni los métodos del player. El acceso al player exige un script inyectado en **main world**.
- WXT 0.21.4 **no** registra los unlisted scripts en `web_accessible_resources`; hay que declararlos a mano o la inyección falla.
- MV3 + `host_permissions` permite llamar a la API de Gemini desde el service worker sin CORS ni proxy propio.
- `gemini-2.5-flash` está restringido por acceso; `gemini-3.5-flash-lite` es el modelo recomendado para traducción de alto volumen.

## Goals / Non-Goals

**Goals:**
- Obtener la **pista completa** de subtítulos de un vídeo, sin gestionar PO tokens ni credenciales de YouTube.
- Adaptar la pista al idioma destino con un **presupuesto de palabras por cue** verificable, con degradación segura al texto original.
- Mostrar la pista adaptada **sincronizada** con el vídeo, sin modificar el audio ni el estado del reproductor.
- Dejar la arquitectura prepared para los dos motores de TTS (cambio posterior) sin reescribir el pipeline.
- Coste por defecto ≈ $0,006 por cada 10 min de vídeo.

**Non-Goals (cambios posteriores previstos):**
- Locución con TTS (`speechSynthesis` y Gemini TTS), retimificación y ducking de volumen. Se cubren en `add-browser-tts-engine` y `add-gemini-tts-engine`.
- Clonación de voz, multi-speaker, detección de hablantes.
- Transcripción de audio para vídeos sin subtítulos.
- Modificación del vídeo, subida de contenido o integración con la cuenta de YouTube.
- App móvil, panel lateral, sincronización de configuración entre dispositivos.

## Decisions

### D1 · Los captions se obtienen interceptando, no descargando
**Qué:** el script de main world parchea `window.fetch` y `XMLHttpRequest` y captura el cuerpo de la respuesta a `/api/timedtext`, reescribiendo la URL para forzar `fmt=json3`.

**Por qué:** la petición del reproductor ya viaja con la atestación válida. Es la única vía que devuelve datos sin gestionar tokens.

**Alternativas descartadas:**
- `fetch` directo al `baseUrl` del player response → HTTP 200 con cuerpo vacío. Solo sirve para obtener la *lista* de pistas.
- Pedir un PO token al servicio de BotGuard → frágil, atado a la versión del player y al estado del navegador; además es material de evasión de medidas anti-bot, que no vamos a construir.
- Scrapear el panel de transcripción del DOM → la lista está virtualizada, no escala a vídeos largos y obliga a hacer scroll. Se conserva únicamente como red de seguridad.

**Consecuencia:** la extracción depende de que la URL parcheada siga siendo válida para el player. De ahí el requisito de `fmt=json3` **y** el fallback a `srv3`/`vtt` en los parsers.

### D2 · Dos mundos, un puente por CustomEvent
**Qué:** `yt-main-world.ts` (unlisted script, main world) para todo lo que toca la red y el player; `youtube.content.ts` (isolated world) para la coordinación; el paso de datos por `CustomEvent` sobre `window`.

**Por qué:** el isolated world comparte el DOM pero no el contexto JS de la página. El main world sí puede ver `window.fetch`, `ytInitialPlayerResponse` y los métodos del player.

**Alternativas descartadas:**
- `window.postMessage` → payload más voluminoso y menos tipado; además es el canal que usan otras extensiones, con lo que el riesgo de confusiones es mayor.
- Exponer una variable en `window` → el contenido sería accesible a la página, que es justo lo que no queremos con los datos de la API.
- Content script con `world: 'MAIN'` → WXT lo soporta de forma nativa en Chromium, pero **no en Firefox** (Manifest V2). Como el objetivo incluye Firefox, se usa `injectScript`.

**Consecuencia:** el bridge valida la forma del payload antes de procesarlo, y el content script nunca acepta cues de un evento que no sea del bridge.

### D3 · El idioma se pide explícitamente al player
**Qué:** antes de esperar la intercepción, se ejecuta `unloadModule('captions')` y `setOption('captions', 'track', { languageCode })` con el idioma fuente detectado, para forzar la petición.

**Por qué:** si el navegador está en español y el vídeo tiene pista en inglés, el player pediría la pista en español y la intercepción solo capturaría esa.

**Alternativas descartadas:** esperar a que el player pida captions por su cuenta → no es determinista: depende del idioma del navegador y de si el usuario tiene los captions activados.

**Consecuencia:** `translationLanguage` se usa como ruta secundaria (0.5) para obtener el idioma fuente cuando el vídeo solo ofrece una traducción automática.

### D4 · json3 como formato principal
**Qué:** se parchea la URL para `fmt=json3` y se parsea `events[].segs[]`.

**Por qué:** es el único formato que da marcas de tiempo por segmento, que es lo que permite medir la duración de cada cue y calcular el presupuesto. `srv3`/`srv1` traen la duración por cue pero no por segmento; `vtt` es legible pero pesado.

**Alternativas descartadas:** parsear `srv3` (XML) como formato principal → menos granular; `vtt` → mismo problema.

**Consecuencia:** los parsers de `srv3`, `srv1` y `vtt` se implementan igual, como degradación, y el sistema **no asume SRT** porque el endpoint no lo devuelve.

### D5 · La completitud se verifica, no se supone
**Qué:** `verifyCompleteness(cues, lengthSeconds)` contrasta número de cues, final de pista, cobertura temporal y hueco máximo; si algo falla, se escala de ruta.

**Por qué:** la ruta de intercepción puede devolver una pista parcial (por ejemplo, solo la primera tanda de cues) sin ningún error visible. Aceptarla en silencio produce un doblaje que se corta a mitad de vídeo.

**Alternativas descartadas:** fiarse del `Content-Length` o de un campo del propio body → la respuesta no expone la duración total esperada de forma fiable.

**Consecuencia:** los umbrales (50 cues, 75 % de la duración, 40 % de cobertura, 30 s de hueco máximo) son **heurísticos**, no exactos: un ASR con silencios largos puede superarlos legítimamente. Por eso el verificador escala de ruta en vez de descartar.

### D6 · El presupuesto de palabras es un número, no un adjetivo
**Qué:** cada cue se envía al modelo con `budgetWords = floor(duracion × 2,9 × 0,92)` y la instrucción dice "no puedes superarlo", con un orden explícito de recorte.

**Por qué:** "recorta drásticamente" no es verificable ni reproducible: el modelo recorta lo que le parece y el resultado es errático. Un número se puede validar, reintentar y auditar.

**Alternativas descartadas:**
- Traducir literal y comprimir después (dos llamadas) → duplica el coste y la latencia; la compresión sobre la traducción literal es tan difícil como hacerla de golpe.
- Fijar un ratio global de compresión → ignora que un cue de 1 s y uno de 20 s toleran recortes muy distintos.
- Traducir al español sin comprimir y dejar que el motor de TTS acelerase → acelerar hasta 1,5× produce un tono inteligible pero artificial, y en el TTS del navegador ni siquiera es controlable.

**Consecuencia:** el prompt permite **fusionar** cues contiguos (devolviendo los índices originales en `source`) y **dividir** cues largos. Es la palanca que hace que el presupuesto sea alcanzable sin mutilar el contenido.

### D7 · Structured output en lugar de parsear texto libre
**Qué:** `generationConfig.responseMimeType: "application/json"` con `responseSchema` que exige `start`, `end`, `text`, `words` y `source`.

**Por qué:** con `responseSchema` el modelo no puede devolver una estructura que rompa el parser, y `start`/`end` propios del modelo permiten **fusionar cues**, que es imposible si el contrato obliga a devolver un timestamps por cue de origen.

**Alternativas descartadas:** parsear JSON a mano desde un texto con markdown; instruir "devuelve solo JSON" sin schema → ambos fallan de forma intermitente y obligan a reintentar a ciegas.

**Consecuencia:** la validación de cliente se centra en el **contenido** (presupuesto, monotonía, solapes), no en la forma.

### D8 · Ventanas de 120 s con look-ahead de 45 s
**Qué:** la pista se divide en ventanas de 120 s; cada una se pide cuando la reproducción entra en los 45 s previos a su inicio. Cada ventana incluye como contexto las últimas líneas de la anterior.

**Por qué:** con look-ahead, el doblaje (o el subtítulo) empieza ~20 s después de abrir el vídeo, no al final. Y el hueco de 45 s es el margen de reintento: si una llamada falla, se reintenta sin que se note.

**Alternativas descartadas:**
- Traducir la pista entera de una vez → sin entrega progresiva (el usuario espera minutos sin nada), y un fallo obliga a repetir todo.
- Ventanas de 30 s → demasiadas llamadas, peor contexto para el modelo y más coste por overhead de sistema.
- Ventanas de 5 min → el primer bloque tardaría demasiado en estar listo.

**Consecuencia:** el contexto de cola del bloque anterior es necesario: sin él el modelo corta frases a la mitad en la frontera de la ventana.

### D9 · Un puntero monótono, no igualdad de timestamps
**Qué:** `DubClock` mantiene un índice que solo avanza; una línea se activa cuando `t >= start` y no ha sido activada. El reloj es `requestVideoFrameCallback` con fallback a `setInterval(100)`.

**Por qué:** `currentTime == cue.start` no se cumple nunca con flotantes, y `timeupdate` dispara a ~4 Hz (250 ms de error). Con puntero + rVFC el error baja a <150 ms y no hay frames perdidos entre cues.

**Alternativas descartadas:** `setInterval` a 50 ms → más gasto de CPU para una mejora marginal; comparar con tolerancia (`|t - start| < 0.2`) → se pierde la línea cuando el sampling cae justo fuera de la ventana.

**Consecuencia:** el motor necesita una heurística de "línea vencida" (si `t > start + tolerancia` y la línea aún no se ha activado, se descarta) para que un retraso de entrega no provoque una ráfaga de líneas atrasadas.

### D10 · La API key solo existe en el service worker
**Qué:** la key vive en `storage.local` y la consume exclusivamente `background.ts`; el content script pide traducciones por mensaje.

**Por qué:** MV3 + `host_permissions` salta el CORS, así que no hace falta proxy. Y si la key vive solo en el worker, un script de la página no tiene forma de leerla: `storage` no es accesible desde el contexto de la página.

**Alternativas descartadas:**
- Proxy propio (servidor) → añade infraestructura, coste y un punto de fuga de la key.
- Llamar a Gemini desde el content script → la key tendría que estar disponible en el isolated world, y una página con XSS o una extensión con Broadly-scoped permission podría leerla.
- `storage.sync` para la key → los datos sincronizados salen del dispositivo; la key no debería viajar.

**Consecuencia:** la interfaz tiene que distinguir "falta la key" de "falló la API", y ofrecer un acceso directo a la configuración.

### D11 · La pantalla de subtítulos es una shadow root, no un elemento del DOM de YouTube
**Qué:** el overlay se monta con `createShadowRootUi` de WXT, en su propio contenedor sobre el reproductor.

**Por qué:** YouTube usa CSS con `!important`, selectores por atributo y un layout en flex/grid muy frágil. Un `div` inyectado en el DOM del player acaba absorbido o deformado.

**Alternativas descartadas:** reutilizar el visor de captions nativo de YouTube → habría que escribir en su estado interno, que no es estable; superponer con `position: fixed` en el body → el z-index compite con los modales de YouTube.

**Consecuencia:** el overlay debe reposicionarse cuando cambia el tamaño de la ventana o el layout del player.

## Risks / Trade-offs

- **[YouTube cambia el mecanismo de carga de captions]** → es el riesgo principal. Mitigación en capas: hook sobre `fetch` **y** `XHR`, cuatro rutas de extracción, verificación de completitud, y un mensaje explícito al usuario si ninguna funciona. El diseño asume que el hook puede romperse mañana.

- **[El PO token se invalida o el player usa otro host]** → el parcheo de URL conserva todos los parámetros originales y solo añade `fmt`, de modo que una futura URL firmada sigue funcionando. Si el player migrara a otro endpoint, habría que actualizar el predicado `/api/timedtext` en un único sitio.

- **[El modelo se pasa del presupuesto de palabras]** → validación en cliente, un reintento con instrucción más estricta y degradación a texto original. Nunca se entrega una línea que no cabe.

- **[La calidad del ASR en inglés es baja]** → se distingue la pista manual de la autogenerada en el estado visible, para que el usuario sepa qué espera del doblaje.

- **[El service worker de MV3 se suspende]** → las peticiones en vuelo mantienen vivo al worker mientras dura la promesa, y el content script reintenta ante un error de "no receiver" o de contexto invalidado.

- **[Cuota de `storage.local` (5 MB)** → los cues se cachean por vídeo y el guion adaptado por vídeo + versión de prompt. No se cachea audio (queda fuera de este cambio). Si una pista muy larga saturara la cuota, se recurre a `storage.session`.

- **[El modelo de la API cambia o se renombra]** → se usa `gemini-3.5-flash-lite`, que es el alias estable recomendado, y la versión de prompt cacheada incluye la identidad del modelo.

- **[Trade-off: entrega progresiva frente a coherencia global]** → las ventanas se traducen de forma independiente, así que una frase que cruza la frontera puede sonar partida. Se mitiga con el contexto de cola; se acepta porque la alternativa (traducir todo antes de empezar) arruina la experiencia.

- **[Trade-off: heurísticas de completitud frente a falsos positivos]** → un ASR con silencios largos puede superar el umbral de hueco máximo. Se prefiere un falso positivo (escalar de ruta) a un falso negativo (pista truncada aceptada).

## Migration Plan

No hay migración de datos: el proyecto está en fase de scaffolding y no hay usuarios.

Despliegue: `wxt build` para Chromium, `wxt build -b firefox` para Firefox. Ambas salidas se cargan como extensiones no empaquetadas en `chrome://extensions` / `about:debugging`.

Rollback: revertir el commit. Como la extensión no escribe nada en el servidor de YouTube ni comparte estado fuera del perfil del navegador, no hay estado que limpiar. La caché en `storage.local` se puede vaciar desde el popup.

## Open Questions

- **Calibración del WPS (2,9 palabras/s)**: es una estimación para `rate` 1,1 en español. Conviene medirla empíricamente con las voces reales del navegador antes de fijar el margen de 0,92; si resulta demasiado holgada, la traducción deja más contenido del que cabe y el doblaje deja huecos.
- **Modelo TTS a futuro**: la elección entre `speechSynthesis` y Gemini TTS condiciona la lógica de sincronización (duración desconocida frente a duración exacta). Se decide en el cambio correspondiente, pero el contrato `DubLine` (con `start`, `end`, `words` y `source`) ya está preparado para ambos.
- **Idiomas de destino**: la lista inicial cubre español en tres variantes, portugués de Brasil e inglés de EE. UU. Añadir idiomas es una entrada más en el popup más una variante en el prompt, pero conviene validar antes que el modelo mantenga la calidad al pasar a idiomas con poco soporte en sus datos.
- **Firefox como objetivo de primera clase**: `injectScript` funciona, pero `speechSynthesis` en Firefox tiene menos voces y un comportamiento distinto. El soporte de Firefox de este cambio es de compilación y de manifiesto; el comportamiento del TTS se decidirá con el motor de locución.
