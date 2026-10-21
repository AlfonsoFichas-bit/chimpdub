## ADDED Requirements

### Requirement: La pista de subtítulos se obtiene interceptando la petición del reproductor
El sistema SHALL obtener la pista de subtítulos interceptando la petición de red que el reproductor de YouTube realiza contra `/api/timedtext`. El sistema NO SHALL emitir una petición propia e independiente para descargar los subtítulos, y NO SHALL depender de la disponibilidad de un token propio.

#### Scenario: El reproductor solicita los captions y la extensión los captura
- **WHEN** el reproductor de YouTube realiza una petición a una URL que contiene `/api/timedtext`
- **THEN** la extensión intercepta la petición, captura el cuerpo de la respuesta y lo expone al content script

#### Scenario: Una petición independiente sin atestación no se usa como fuente
- **WHEN** el endpoint `/api/timedtext` responde con código 200 y cuerpo vacío al ser consultado sin la atestación del reproductor
- **THEN** el sistema interpreta el cuerpo vacío como fallo de esa ruta y escala a la siguiente ruta de la cascada, sin dar por agotada la extracción

### Requirement: La URL interceptada se parchea para solicitar el formato json3
El sistema SHALL reescribir la URL de la petición interceptada para incluir el parámetro `fmt=json3` antes de que la petición salga de la página, de modo que la respuesta contenga marcas de tiempo por segmento.

#### Scenario: Petición parcheada con formato json3
- **WHEN** la extensión intercepta una petición a `/api/timedtext` que no incluye `fmt`
- **THEN** la petición que sale hacia la red incluye `fmt=json3` y el cuerpo capturado se interpreta como json3

#### Scenario: Se preservan los demás parámetros de la URL
- **WHEN** la URL interceptada contiene parámetros como `lang`, `vss_id`, `kind` o `tlang`
- **THEN** la URL parcheada los conserva íntegros y solo añade o sustituye `fmt`

### Requirement: Se interceptan tanto fetch como XMLHttpRequest
El sistema SHALL parchear tanto `window.fetch` como `XMLHttpRequest.prototype.open`/`send` para detectar peticiones a `/api/timedtext`, de modo que la extracción no dependa de la técnica de red que use la versión vigente del reproductor.

#### Scenario: Petición realizada con XMLHttpRequest
- **WHEN** el reproductor solicita los captions mediante `XMLHttpRequest`
- **THEN** la extensión detecta la URL en `open`, lee `responseText` en el evento `load` y expone el cuerpo capturado

#### Scenario: Peticiones ajenas a los captions no se tocan
- **WHEN** la página realiza una petición que no contiene `/api/timedtext`
- **THEN** la extensión la ejecuta sin modificar y sin emitir ningún evento

### Requirement: La pista completa se obtiene en una sola respuesta
El sistema SHALL tratar el cuerpo capturado como la pista completa del vídeo y NO SHALL paginar, recortar ni solicitar subtítulos cue a cue.

#### Scenario: Una respuesta contiene todos los cues del vídeo
- **WHEN** la respuesta interceptada contiene N cues que cubren el vídeo de principio a fin
- **THEN** el sistema los expone como la pista completa sin realizar ninguna petición adicional de captions

### Requirement: Se soportan varios formatos de respuesta de captions
El sistema SHALL ser capaz de interpretar los formatos `json3`, `srv3`, `srv1` y `vtt`. El sistema NO SHALL asumir que el endpoint devuelve SRT.

#### Scenario: Respuesta en json3
- **WHEN** el cuerpo capturado es JSON con `events[].segs[].utf8`, `tStartMs` y `dDurationMs`
- **THEN** el sistema produce un cue por segmento con inicio y fin en segundos

#### Scenario: Respuesta en srv3 o srv1
- **WHEN** el cuerpo capturado es XML con elementos `text` que llevan atributos de inicio y duración
- **THEN** el sistema produce cues equivalentes con inicio y fin en segundos

#### Scenario: Respuesta en vtt
- **WHEN** el cuerpo capturado es WebVTT
- **THEN** el sistema produce cues equivalentes, descartando las marcas de tiempo inline

#### Scenario: Formato desconocido
- **WHEN** el cuerpo capturado no corresponde a ninguno de los formatos soportados
- **THEN** el sistema informa del fallo y escala a la siguiente ruta de la cascada

### Requirement: El texto de los captions se normaliza antes de usarse
El sistema SHALL normalizar el texto de los cues antes de cualquier uso posterior, eliminando entidades HTML, tags de formato (`<c>`, `<i>`, `<b>`, `<font>`), marcas de tiempo inline de VTT, saltos de línea hard-coded y espacios múltiples, y resolviendo entidades como `&amp;`, `&#39;`, `&quot;` y `&nbsp;` a sus caracteres.

#### Scenario: Cue con tags de formato y entidades
- **WHEN** un cue contiene `<c.colorE5E5E5>Hola &amp; adi&oacute;s</c>`
- **THEN** el texto normalizado es `Hola & adiós` sin tags ni entidades

#### Scenario: Cue con marca de tiempo inline de VTT
- **WHEN** un cue contiene `<00:00:01.000>` dentro del texto
- **THEN** la marca de tiempo desaparece del texto normalizado

### Requirement: Se fuerza la carga de la pista en el idioma fuente
El sistema SHALL forzar al reproductor a solicitar la pista de subtítulos del idioma fuente mediante la API del player, especificando explícitamente el `languageCode` y descargando antes el módulo de captions.

#### Scenario: Se pide explícitamente el idioma fuente
- **WHEN** la extracción comienza y el reproductor no ha solicitado captions para el idioma fuente
- **THEN** la extensión ejecuta `unloadModule('captions')` y `setOption('captions', 'track', { languageCode })` con el idioma fuente detectado

#### Scenario: El idioma del navegador no coincide con el de la pista
- **WHEN** el navegador está configurado en español y la pista disponible está en inglés
- **THEN** la extensión solicita explícitamente la pista en inglés en lugar de asumir el idioma del reproductor

### Requirement: Se comprueba la completitud de la pista antes de aceptarla
El sistema SHALL verificar la completitud de la pista extraída antes de exponerla como válida, contrastando el número de cues, el final de la pista, la cobertura temporal y el hueco máximo entre cues contra la duración del vídeo obtenida del player response.

#### Scenario: Pista completa
- **WHEN** la pista tiene más de 50 cues, su último cue alcanza al menos el 75 % de la duración del vídeo, la cobertura temporal supera el 40 % y ningún hueco entre cues supera los 30 s
- **THEN** el sistema marca la pista como completa y la acepta

#### Scenario: Pista truncada
- **WHEN** la pista termina antes del 75 % de la duración del vídeo o presenta un hueco superior a 30 s
- **THEN** el sistema marca la pista como incompleta y escala a la siguiente ruta de la cascada

#### Scenario: Pista que no cubre el vídeo
- **WHEN** la cobertura temporal de la pista es inferior al 40 %
- **THEN** el sistema marca la pista como incompleta y escala a la siguiente ruta de la cascada

### Requirement: La extracción escala por una cascada de rutas
El sistema SHALL intentar la extracción por, al menos, las siguientes rutas en orden: (0) intercepción de la petición del reproductor con parcheo de formato; (1) `baseUrl` de captions obtenida del player response y consultada directamente; (2) panel de transcripción del DOM; (3) lectura del `ytInitialPlayerResponse` del HTML de la página. El sistema SHALL escalar a la siguiente ruta cuando la actual no devuelva una pista completa, y SHALL informar al usuario si ninguna ruta funciona.

#### Scenario: La ruta primaria funciona
- **WHEN** la petición interceptada devuelve una pista que supera la verificación de completitud
- **THEN** el sistema no consulta ninguna ruta adicional

#### Scenario: La ruta primaria devuelve cuerpo vacío
- **WHEN** la petición interceptada devuelve un cuerpo vacío
- **THEN** el sistema escala a la ruta siguiente y reintenta la extracción

#### Scenario: Ninguna ruta funciona
- **WHEN** todas las rutas han fallado
- **THEN** el sistema muestra un mensaje explícito de que no hay subtítulos disponibles y no deja el pipeline en silencio

### Requirement: Se prefiere la mejor pista disponible
Cuando existan varias pistas, el sistema SHALL preferir la pista manual sobre la autogenerada (ASR) y la pista en el idioma fuente del audio sobre la traducción automática de YouTube.

#### Scenario: Hay pista manual y pista ASR en el mismo idioma
- **WHEN** el vídeo ofrece una pista manual y una pista autogenerada en el mismo idioma
- **THEN** el sistema elige la pista manual

#### Scenario: La única pista disponible es una traducción automática
- **WHEN** la única pista disponible es la traducción automática de YouTube al idioma destino
- **THEN** el sistema la acepta como última instancia y la marca como traducción automática en el estado visible al usuario

### Requirement: Los cues extraídos se cachean por vídeo e idioma
El sistema SHALL almacenar los cues normalizados en almacenamiento local, indexados por identificador de vídeo e idioma de origen, y SHALL reutilizarlos en lugar de repetir la extracción cuando ya están disponibles.

#### Scenario: Reutilización de la caché
- **WHEN** se abre de nuevo un vídeo cuyos cues ya están cacheados
- **THEN** el sistema los reutiliza y no vuelve a interceptar la petición de captions

#### Scenario: Caché de un vídeo distinto
- **WHEN** se abre un vídeo con un identificador distinto
- **THEN** el sistema no reutiliza los cues cacheados del vídeo anterior

### Requirement: La extensión comunica las situaciones sin subtítulos
El sistema SHALL distinguir e informar al usuario de los casos en que no hay extracción posible: vídeo sin ninguna pista de subtítulos, vídeo restringido o privado sin sesión iniciada, y vídeo cuyo idioma ya es el idioma destino.

#### Scenario: Vídeo sin subtítulos
- **WHEN** el vídeo no ofrece ninguna pista de subtítulos
- **THEN** el sistema informa de que no hay subtítulos disponibles en lugar de mostrar un error genérico

#### Scenario: Vídeo ya en el idioma destino
- **WHEN** la única pista disponible está en el idioma destino configurado
- **THEN** el sistema informa de que el vídeo ya está en ese idioma y no lanza la traducción
