## 1. Fase 0 · Andamiaje, manifiesto y base de tipos

- [ ] 1.1 Escribir el bloque `manifest` en `wxt.config.ts`: `permissions: ['storage', 'tabs']`, `host_permissions: ['https://generativelanguage.googleapis.com/*', '*://*.youtube.com/*']` y `web_accessible_resources` con `yt-main-world.js` para `*://*.youtube.com/*`
- [ ] 1.2 Eliminar `entrypoints/content.ts` (stub de `*.google.com`) y crear `entrypoints/youtube.content.ts` con `matches: ['*://*.youtube.com/*']` y `run_at: 'document_start'`
- [ ] 1.3 Crear `entrypoints/yt-main-world.ts` como unlisted script (`defineUnlistedScript`) con un esqueleto que escuche el evento del bridge
- [ ] 1.4 Definir en `utils/types.ts` los tipos `Cue`, `DubLine`, `CaptionTrack`, `ExtractedTrack`, `WindowSize` y `DubConfig`
- [ ] 1.5 Crear `utils/storage.ts` con `storage.defineItem` para `local:apiKey`, `local:config`, `local:captions:<videoId>:<lang>` y `local:dub:<videoId>:<lang>:<promptVersion>`
- [ ] 1.6 Crear `utils/bridge.ts` con los nombres de los `CustomEvent`, los tipos de payload y los helpers `emit`/`on` con validación de forma
- [ ] 1.7 Verificar que `npm run build` genera un manifiesto válido y que `tsc --noEmit` pasa

## 2. Fase 0 · Popup de configuración (SolidJS)

- [ ] 2.1 Sustituir el contador de `entrypoints/popup/App.tsx` por el formulario de configuración: API key (input enmascarado con botón de revelar), idioma destino, nivel de compresión, idioma origen
- [ ] 2.2 Implementar el guardado de la API key en `local:apiKey` con confirmación visual en el popup
- [ ] 2.3 Implementar "probar conexión": llamada mínima a `generativelanguage.googleapis.com` desde el background, con mensaje de éxito o de error (key inválida, sin permisos, cuota)
- [ ] 2.4 Implementar el selector de idioma destino con la lista `es-ES`, `es-MX`, `es-419`, `pt-BR`, `en-US` y rechazo de valores no soportados
- [ ] 2.5 Persistir el idioma destino y el nivel de compresión en `local:config` y verificar que persisten tras reiniciar el navegador
- [ ] 2.6 Mostrar en el popup el estado de la última traducción (etapa, ventanas procesadas, error) leyéndolo del estado compartido

## 3. Fase 1 · Extracción de captions (fase crítica)

- [ ] 3.1 Implementar en `yt-main-world.ts` el parcheo de `window.fetch`: detectar URLs con `/api/timedtext`, reescribir la URL con `fmt=json3` conservando el resto de parámetros, y emitir el evento con `res.clone().text()`
- [ ] 3.2 Implementar el parcheo de `XMLHttpRequest.prototype.open`/`send` con el mismo predicado, leyendo `responseText` en el evento `load`
- [ ] 3.3 Implementar en `yt-main-world.ts` las funciones de control del player: `getPlayerResponse()`, `getDuration()`, `getCaptionTracks()` y `setCaptionTrack(languageCode)`
- [ ] 3.4 Implementar `composables/captions/parse-json3.ts`: recorrer `events[].segs[]`, calcular `start`/`end` en segundos y concatenar los `utf8` de cada evento
- [ ] 3.5 Implementar `composables/captions/parse-xml.ts` para `srv3` (`start`/`dur`) y `srv1` (`t`/`d`)
- [ ] 3.6 Implementar el parser de `vtt` con eliminación de timestamps inline y de la cabecera
- [ ] 3.7 Implementar `composables/captions/clean.ts`: entidades HTML, tags `<c>`/`<i>`/`<b>`/`<font>`, timestamps inline, saltos de línea y espacios múltiples
- [ ] 3.8 Implementar `composables/captions/verify.ts` con `verifyCompleteness(cues, durationSec)`: cue count, `lastEnd` vs 75 % de duración, cobertura vs 40 % y hueco máximo vs 30 s
- [ ] 3.9 Implementar `composables/captions/extract.ts` con la cascada de rutas (0 intercepción → 1 baseUrl directo → 2 panel de transcripción → 3 `ytInitialPlayerResponse` del HTML) y la escalada por verificación incompleta
- [ ] 3.10 Implementar la selección de pista por taxonomía de `vssId` (manual > ASR, idioma fuente > traducción automática) y la ruta secundaria con `translationLanguage`
- [ ] 3.11 Cachear los cues normalizados en `local:captions:<videoId>:<lang>` y reutilizarlos si ya existen
- [ ] 3.12 Detectar y comunicar los casos sin subtítulos: sin ninguna pista, vídeo restringido sin sesión, y vídeo ya en el idioma destino
- [ ] 3.13 **Punto de verificación**: sobre un vídeo real de ≥ 10 min, comprobar que la pista extraída tiene más de 50 cues, que el último cue alcanza ≥ 75 % de la duración y que no hay huecos > 30 s

## 4. Fase 2 · Traducción y adaptación

- [ ] 4.1 Implementar `composables/budget.ts` con `budgetWords(durationSec)` = `floor(dur × 2,9 × 0,92)` y mínimo de 1 palabra
- [ ] 4.2 Implementar `composables/ai/prompt.ts` con el system instruction: traducir al español natural, respetar el presupuesto, orden de recorte (muletillas → redundancias → adverbios), conservar nombres/cifras/fechas/negaciones, permitir fusión y división de cues, y devolver JSON sin etiquetas
- [ ] 4.3 Implementar `composables/ai/translate.ts` en el background: `POST /v1beta/models/gemini-3.5-flash-lite:generateContent` con `x-goog-api-key`, `responseMimeType: 'application/json'` y el `responseSchema` de `start`/`end`/`text`/`words`/`source`
- [ ] 4.4 Implementar la validación del resultado: presupuesto por línea, `end > start`, monotonía, no solape y texto no vacío
- [ ] 4.5 Implementar el reintento con instrucción reforzada y la degradación final al texto original del cue
- [ ] 4.6 Implementar el troceado en ventanas de 120 s con look-ahead de 45 s y el contexto de cola de ~10 s de la ventana anterior
- [ ] 4.7 Implementar la entrega progresiva: cada ventana notificada al content script en cuanto está lista, sin bloquear las siguientes
- [ ] 4.8 Cachear el guion adaptado en `local:dub:<videoId>:<lang>:<promptVersion>` e invalidarlo al cambiar la versión de prompt
- [ ] 4.9 Implementar el manejo de errores de la API: sin API key, key inválida, cuota agotada y error transitorio con reintentos y espera creciente
- [ ] 4.10 **Punto de verificación**: en un bloque de 2 min, el 100 % de las líneas respetan `words ≤ budgetWords` tras el reintento, y el primer bloque está disponible en menos de 15 s

## 5. Fase 3 · Motor de sincronización y overlay

- [ ] 5.1 Implementar `composables/player/clock.ts` con `requestVideoFrameCallback` y fallback a `setInterval(100)`, exponiendo el tiempo de reproducción
- [ ] 5.2 Implementar el puntero monótono de cues: avance por índice, marca de líneas activadas, detección de líneas vencidas y descarte de líneas pasadas
- [ ] 5.3 Implementar la detección de seek (salto > 1,5 s) con reposicionamiento del puntero
- [ ] 5.4 Implementar el rearme del puntero en pausa y reanudación
- [ ] 5.5 Implementar `composables/player/controller.ts` con los accesos al player vía bridge: `getPlayerResponse`, `getDuration`, `getCaptionTracks`, `setCaptionTrack`, `getVolume`, `setVolume`, `unloadModule`
- [ ] 5.6 Montar el overlay con `createShadowRootUi` sobre el reproductor, con estilos propios en el shadow root
- [ ] 5.7 Renderizar la línea activa sobre el vídeo con error < 150 ms respecto al cue de origen, respetando la fusión de cues (`start` del primero, `end` del último)
- [ ] 5.8 Implementar el panel de estado del overlay: etapa actual (extrayendo, adaptando, listo, error), ventana en curso sobre total e idioma destino
- [ ] 5.9 Implementar el botón de reintento ante error irrecuperable y el acceso directo a la configuración cuando falta la API key
- [ ] 5.10 Implementar el reinicio del pipeline ante navegación interna de YouTube con `wxt:locationchange`: cancelar trabajo, limpiar estado y reextraer
- [ ] 5.11 **Punto de verificación**: los subtítulos en español se muestran sincronizados con error < 150 ms y sobreviven a pausa, seek y cambio de vídeo sin desincronizarse

## 6. Verificación, builds y documentación

- [ ] 6.1 Comprobar que la extensión no modifica volumen, silencio ni velocidad de reproducción mientras muestra los subtítulos
- [ ] 6.2 Comprobar que la API key no es accesible desde el contexto de la página de YouTube
- [ ] 6.3 Comprobar que `npm run build` (Chromium) y `npm run build:firefox` generan manifiestos válidos, con `web_accessible_resources` en array plano para Firefox
- [ ] 6.4 Verificar el coste real de una traducción de 10 min con la API key del usuario y contrastarlo con la estimación de $0,006
- [ ] 6.5 Actualizar `docs/plan.md` con el estado de las fases 0 a 3 y el alcance de los cambios de TTS posteriores
- [ ] 6.6 Documentar en el README cómo obtener la API key, los permisos requeridos y las limitaciones conocidas
