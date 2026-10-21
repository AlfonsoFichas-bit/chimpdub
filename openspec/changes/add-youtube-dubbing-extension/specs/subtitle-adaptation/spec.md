## ADDED Requirements

### Requirement: Cada cue recibe un presupuesto de palabras derivado de su duración
El sistema SHALL calcular, para cada cue de origen, un presupuesto de palabras calculado como `floor(duracionSegundos × palabrasPorSegundo × margen)`, con un objetivo de 2,9 palabras por segundo y un margen de 0,92, y SHALL incluir ese presupuesto en la petición al modelo.

#### Scenario: Cálculo del presupuesto
- **WHEN** un cue dura 3,0 segundos
- **THEN** su presupuesto es de 8 palabras y ese valor se envía al modelo junto con el texto del cue

#### Scenario: Cue de duración cero
- **WHEN** un cue tiene duración cero o desconocida
- **THEN** el presupuesto se calcula con un mínimo de una palabra para no generar una línea inválida

### Requirement: El modelo no puede superar el presupuesto de palabras
El sistema SHALL validar en cliente que ninguna línea adaptada supere el presupuesto de palabras de su cue de origen. Si la validación falla, el sistema SHALL reintentar una vez con una instrucción más estricta y, si el reintento también falla, SHALL usar el texto original de ese cue.

#### Scenario: La primera respuesta supera el presupuesto
- **WHEN** una línea devuelta tiene más palabras que su presupuesto
- **THEN** el sistema descarta la respuesta y reintenta el bloque con una instrucción más restrictiva

#### Scenario: El reintento también incumple
- **WHEN** la segunda respuesta sigue superando el presupuesto
- **THEN** el sistema usa el texto original del cue correspondiente y el resto del bloque se conserva

#### Scenario: Todas las líneas cumplen
- **WHEN** todas las líneas del bloque respetan su presupuesto
- **THEN** el sistema acepta el bloque y lo entrega al motor de reproducción

### Requirement: La adaptación prioriza el significado sobre el ritmo
El sistema SHALL instruir al modelo para que recorte, por este orden, muletillas y relleno, redundancias y repeticiones, y adverbios y adjetivos prescindibles, y para que conserve siempre los nombres propios, las cifras, las fechas, los datos concretos, las negaciones y el sentido general de la frase.

#### Scenario: Cue con relleno y un dato concreto
- **WHEN** el cue original dice "Bueno, o sea, el experimento de 1998 dio 42 resultados"
- **THEN** la adaptación esperada elimina el relleno y conserva "1998" y "42"

#### Scenario: El modelo propone eliminar una cifra
- **WHEN** una adaptación propuesta suprime un dato concreto del original
- **THEN** el sistema la considera no conforme con la política de adaptación

### Requirement: La salida del modelo es JSON estructurado y validable
El sistema SHALL solicitar la salida del modelo mediante la Interactions API (`POST /v1beta/interactions`) con `response_format: { type: "text", mime_type: "application/json", schema }`, donde `schema` es JSON Schema estándar que define un array de objetos con las propiedades `start`, `end`, `text`, `words` y `source`, y SHALL leer el texto generado del paso `model_output` de la respuesta.

#### Scenario: Respuesta conforme con el esquema
- **WHEN** el modelo devuelve un array de objetos con todas las propiedades requeridas
- **THEN** el sistema lo parsea y lo pasa a la validación de tiempos y presupuesto

#### Scenario: Respuesta no parseable
- **WHEN** la respuesta del modelo no es JSON válido o no respeta el esquema
- **THEN** el sistema la descarta, reintenta el bloque y, si persiste el fallo, degrada a texto original

### Requirement: La llamada al modelo se hace por HTTP directo y sin SDK
El sistema SHALL llamar a la API de Gemini con `fetch` nativo desde el service worker, sin incorporar un SDK de cliente, y SHALL enviar la clave en la cabecera `x-goog-api-key`.

#### Scenario: El bundle no incluye SDK de Gemini
- **WHEN** se inspecciona el bundle generado del service worker
- **THEN** no contiene ninguna librería cliente de la API de Gemini

#### Scenario: La petición se origina en el service worker
- **WHEN** se pide una traducción
- **THEN** la petición a `generativelanguage.googleapis.com` la emite el service worker

### Requirement: El contenido enviado al modelo no se almacena en el servidor
El sistema SHALL enviar `store: false` en la petición, de modo que las interacciones no se retengan en el servidor de Google más allá de la propia respuesta.

#### Scenario: Petición con store desactivado
- **WHEN** se realiza una traducción
- **THEN** el cuerpo de la petición incluye `store: false`

### Requirement: Las líneas adaptadas son monotónicas y no se solapan
El sistema SHALL validar que las líneas adaptadas devueltas tienen `end` mayor que `start`, que sus tiempos son monotónicos crecientes y que ninguna línea se solapa con la anterior. Las líneas inválidas SHALL ser corregidas por el modelo (que devuelve sus propios tiempos) o descartadas.

#### Scenario: Tiempo invertido
- **WHEN** una línea devuelta tiene `end` menor o igual que `start`
- **THEN** el sistema la considera inválida y reintenta el bloque

#### Scenario: Solape entre líneas
- **WHEN** el inicio de una línea es anterior al final de la línea anterior
- **THEN** el sistema considera el bloque inválido y lo reintenta

### Requirement: La adaptación se procesa en ventanas progresivas con look-ahead
El sistema SHALL dividir la pista en ventanas de 120 segundos y SHALL solicitar cada ventana cuando la reproducción entra en los 45 segundos previos a su inicio, de modo que la traducción esté lista antes de que se necesite.

#### Scenario: Primera ventana
- **WHEN** la extracción termina en el segundo 1
- **THEN** el sistema solicita de inmediato la ventana que cubre los primeros 120 segundos, incluyendo como contexto las líneas finales de la ventana anterior

#### Scenario: Ventanas sucesivas
- **WHEN** la reproducción alcanza el segundo 75
- **THEN** el sistema solicita la ventana que empieza en el segundo 120

#### Scenario: Contexto de cola
- **WHEN** el sistema construye la ventana que empieza en el segundo 120
- **THEN** incluye las líneas de la ventana anterior como contexto de lectura, y no como parte de la salida

### Requirement: La entrega es progresiva y no bloquea el resto del vídeo
El sistema SHALL entregar cada ventana adaptada en cuanto está disponible, y el fallo de una ventana NO SHALL impedir la adaptación ni la entrega de las ventanas siguientes.

#### Scenario: La primera ventana está lista antes de que avance la reproducción
- **WHEN** la ventana inicial tarda unos segundos en adaptarse
- **THEN** el motor de reproducción recibe esa ventana y puede mostrar las primeras líneas

#### Scenario: Una ventana intermedia falla
- **WHEN** la ventana que empieza en el segundo 240 agota sus reintentos
- **THEN** esa ventana se degrada a texto original y la ventana 360 se procesa con normalidad

### Requirement: El guion adaptado se cachea con versión de prompt
El sistema SHALL almacenar el guion adaptado indexado por identificador de vídeo, idioma de destino y versión de prompt, y SHALL reutilizarlo mientras la versión de prompt no cambie.

#### Scenario: Reutilización del guion
- **WHEN** se vuelve a abrir un vídeo cuyo guion está cacheado con la versión de prompt vigente
- **THEN** el sistema no vuelve a llamar al modelo

#### Scenario: Invalidación por cambio de prompt
- **WHEN** la versión de prompt cambia
- **THEN** el sistema no reutiliza el guion cacheado y vuelve a adaptarlo

### Requirement: El idioma de destino es configurable
El sistema SHALL permitir configurar el idioma de destino entre al menos `es-ES`, `es-MX`, `es-419`, `pt-BR` y `en-US`, y SHALL incluir el matiz regional correspondiente en la instrucción al modelo y en la selección de voz.

#### Scenario: Idioma no soportado
- **WHEN** se configura un idioma fuera de la lista soportada
- **THEN** el sistema rechaza la configuración y mantiene el idioma previo

### Requirement: La API key se usa únicamente en el service worker
El sistema SHALL almacenar la API key de Gemini en almacenamiento local del navegador y SHALL usarla exclusivamente desde el service worker. La key NO SHALL enviarse a la página de YouTube, NO SHALL incluirse en el bundle y NO SHALL exponerse a ningún script de la página.

#### Scenario: La página intenta leer la configuración
- **WHEN** un script de la página de YouTube inspecciona `window` o el DOM en busca de la key
- **THEN** no encuentra la key en ningún momento de la ejecución

#### Scenario: La petición a Gemini se origina en el service worker
- **WHEN** se pide una traducción
- **THEN** la llamada a `generativelanguage.googleapis.com` la emite el service worker, que añade la cabecera `x-goog-api-key`

### Requirement: Los errores del servicio de traducción se comunican al usuario
El sistema SHALL diferenciar y comunicar al usuario los fallos del servicio: ausencia de API key configurada, key inválida o sin permisos, cuota agotada y error transitorio de red.

#### Scenario: Sin API key configurada
- **WHEN** se intenta adaptar una pista sin API key guardada
- **THEN** el sistema informa de que falta la API key y ofrece abrir la configuración, sin lanzar la petición

#### Scenario: Error transitorio
- **WHEN** la llamada al modelo falla por un error de red
- **THEN** el sistema reintenta con espera creciente y, tras agotar los reintentos, informa del fallo y degrada a texto original
