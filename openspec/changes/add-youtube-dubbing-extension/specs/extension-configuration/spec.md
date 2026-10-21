## ADDED Requirements

### Requirement: El manifiesto declara únicamente los permisos necesarios
El sistema SHALL declarar en el manifiesto los permisos `storage` y `tabs`, los permisos de host `https://generativelanguage.googleapis.com/*` y `*://*.youtube.com/*`, y SHALL content script con coincidencia `*://*.youtube.com/*`.

#### Scenario: Permisos de host declarados
- **WHEN** se inspecciona el manifiesto generado
- **THEN** aparecen el host de la API de Gemini y el de YouTube en `host_permissions`

#### Scenario: No hay permisos de host amplo
- **WHEN** se inspecciona el manifiesto generado
- **THEN** no hay comodines de tipo `*://*://*` ni permisos de host sobre sitios distintos de YouTube y de la API de Gemini

### Requirement: El script de main world se declara como recurso accesible
El sistema SHALL declarar el script de main world en `web_accessible_resources` con la coincidencia de YouTube, porque el generador no los registra automáticamente.

#### Scenario: El script es inyectable desde la página
- **WHEN** el content script intenta inyectar el script de main world
- **THEN** la petición del recurso no es bloqueada por las reglas de recursos accesibles

### Requirement: La API key del usuario se guarda de forma local y persistente
El sistema SHALL almacenar la API key de Gemini en el almacenamiento local del navegador, cifrada por el propio perfil del navegador y fuera del área accesible a la página web. La key NO SHALL guardarse en almacenamiento sincronizado entre dispositivos.

#### Scenario: La key persiste entre sesiones
- **WHEN** el usuario guarda la key y reinicia el navegador
- **THEN** la key sigue configurada

#### Scenario: La key no viaja entre dispositivos
- **WHEN** el perfil del navegador sincroniza sus datos
- **THEN** la API key no forma parte de los datos sincronizados

### Requirement: El popup permite configurar la extensión
El sistema SHALL proveer un popup que permita introducir y validar la API key, elegir el idioma de destino, elegir el nivel de compresión, y consultar el estado de la última traducción.

#### Scenario: Guardar la API key
- **WHEN** el usuario introduce la key y la guarda
- **THEN** el popup confirma el guardado y la key queda almacenada

#### Scenario: Validar la API key
- **WHEN** el usuario pulsa "probar conexión"
- **THEN** el popup realiza una llamada mínima a la API de Gemini y muestra el resultado, sin revelar la key completa

#### Scenario: El campo de la key se muestra enmascarado
- **WHEN** la key está guardada y se abre el popup
- **THEN** el campo la muestra enmascarada y solo la revela bajo una acción explícita

### Requirement: El idioma de destino se persiste
El sistema SHALL persistir el idioma de destino configurado entre sesiones y entre vídeos.

#### Scenario: El idioma sobrevive al reinicio
- **WHEN** el usuario fija `pt-BR` y reinicia el navegador
- **THEN** el idioma destino sigue siendo `pt-BR`

### Requirement: La comunicación entre main world e isolated world usa eventos del DOM
El sistema SHALL transportar datos entre el script de main world y el content script mediante eventos `CustomEvent` disparados sobre `window`, y SHALL validar el origen y la forma del payload antes de procesarlo.

#### Scenario: Captura de los captions en main world
- **WHEN** el script de main world intercepta una respuesta de captions
- **THEN** emite un evento con la URL, el cuerpo y los metadatos de la pista

#### Scenario: Evento con payload inválido
- **WHEN** llega un evento con una forma de payload no reconocida
- **THEN** el content script lo ignora sin lanzar una excepción

#### Scenario: Un script de la página no puede inyectar datos falsos
- **WHEN** un script de la página emite un evento con la misma forma que el del bridge
- **THEN** el content script no lo usa para modificar los cues ya extraídos

### Requirement: El content script arranca antes de que el reproductor cargue
El sistema SHALL registrar el content script con `run_at: "document_start"` e inyectar el script de main world en ese momento, para no perder la primera petición de captions.

#### Scenario: Inyección temprana
- **WHEN** se carga una página de YouTube
- **THEN** el script de main world está instalado antes de que el reproductor envíe su primera petición de captions

### Requirement: El estado del pipeline se comparte entre superficies
El sistema SHALL compartir el estado del pipeline (etapa actual, ventana en curso, totales, error) entre el content script, el popup y el overlay.

#### Scenario: El popup refleja el estado actual
- **WHEN** el usuario abre el popup mientras un vídeo se está adaptando
- **THEN** el popup muestra la etapa y el progreso actuales

#### Scenario: El overlay refleja la configuración
- **WHEN** el usuario cambia el idioma de destino en el popup
- **THEN** el overlay de un vídeo ya abierto refleja el cambio y reinicia la adaptación con el nuevo idioma

### Requirement: La extensión soporta Chromium y Firefox
El sistema SHALL compilar para Chromium y para Firefox, y el build de Firefox SHALL declarar los recursos accesibles con el formato plano que exige Manifest V2.

#### Scenario: Build de Firefox
- **WHEN** se ejecuta el build para Firefox
- **THEN** el manifiesto generado declara `web_accessible_resources` como array plano y los permisos de host como `permissions`

### Requirement: El error de configuración se muestra de forma accionable
El sistema SHALL indicar al usuario qué hacer cuando falta la API key, sin limitarse a mostrar un mensaje de error genérico.

#### Scenario: Falta la API key y el usuario intenta traducir
- **WHEN** el pipeline necesita traducir y no hay API key configurada
- **THEN** la interfaz indica que falta la API key y ofrece un acceso directo a la configuración
