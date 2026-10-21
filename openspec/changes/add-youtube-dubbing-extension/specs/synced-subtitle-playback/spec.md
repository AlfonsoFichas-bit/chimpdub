## ADDED Requirements

### Requirement: La sincronización usa un puntero monótono de cues
El sistema SHALL determinar la línea activa comparando el tiempo de reproducción con los tiempos de las líneas mediante un índice que avanza de forma monótona, y NO SHALL depender de la igualdad exacta entre `currentTime` y el inicio de una línea.

#### Scenario: El tiempo de reproducción pasa por encima del inicio de una línea
- **WHEN** el tiempo de reproducción avanza entre dos frames y cruza el inicio de una línea
- **THEN** esa línea se activa en el siguiente tick, sin necesidad de que `currentTime` coincida exactamente con su inicio

#### Scenario: El puntero nunca retrocede
- **WHEN** el tiempo de reproduction avanza
- **THEN** el índice del puntero solo avanza y las líneas ya superadas no se vuelven a activar

#### Scenario: Líneas ya superadas al arrancar
- **WHEN** el motor se activa con el vídeo en el segundo 10 y hay líneas que han terminado antes de ese momento
- **THEN** esas líneas se marcan como no activadas y no se muestran, y la primera línea activa es la primera que termina después del segundo 10

### Requirement: El reloj se actualiza a frecuencia de frame
El sistema SHALL usar `video.requestVideoFrameCallback` como mecanismo principal de actualización del reloj, y SHALL recurrir a un temporizador de 100 ms cuando esa API no esté disponible. El sistema NO SHALL usar el evento `timeupdate` como mecanismo de sincronización.

#### Scenario: Navegador con requestVideoFrameCallback
- **WHEN** el navegador implementa `requestVideoFrameCallback`
- **THEN** el reloj se actualiza en cada callback de frame de vídeo

#### Scenario: Navegador sin la API
- **WHEN** el navegador no implementa `requestVideoFrameCallback`
- **THEN** el reloj se actualiza mediante un temporizador de 100 ms

### Requirement: Un salto en el reloj se interpreta como seek
El sistema SHALL detectar un cambio brusco del tiempo de reproducción, superior a 1,5 segundos, como un salto del usuario y SHALL repositionar el puntero de cues al cue correspondiente al nuevo tiempo.

#### Scenario: El usuario hace seek hacia delante
- **WHEN** el tiempo de reproducción salta de 30 s a 300 s
- **THEN** el puntero se reposiciona a la línea que contiene el segundo 300 y el resto del vídeo se reproduce desde ahí

#### Scenario: El usuario hace seek hacia atrás
- **WHEN** el tiempo de reproducción salta de 300 s a 30 s
- **THEN** el puntero se reposiciona hacia atrás y las líneas posteriores vuelven a activarse

#### Scenario: Salto natural de la reproducción
- **WHEN** el tiempo de reproducción avanza de forma continua sin saltos superiores a 1,5 s
- **THEN** el sistema no interpreta el avance como seek

### Requirement: La línea mostrada está sincronizada con el vídeo
El sistema SHALL mostrar la línea adaptada correspondiente al tiempo de reproducción con un error inferior a 150 ms respecto al inicio de la línea de origen.

#### Scenario: Reproducción continua
- **WHEN** el vídeo se reproduce de forma continua
- **THEN** la línea visible en cada momento corresponde a la línea de origen activa en ese instante, con una desviación inferior a 150 ms

#### Scenario: Línea sin equivalente de origen
- **WHEN** la adaptación ha fusionado dos cues de origen en una línea
- **THEN** la línea fusionada se muestra desde el inicio del primer cue de origen hasta el final del segundo

### Requirement: Las líneas vencidas se saltan en lugar de acumularse
El sistema SHALL descartar toda línea cuyo final sea anterior al tiempo de reproducción en el momento de activarse, de modo que nunca se acumulen líneas pendientes.

#### Scenario: Retardo de la entrega
- **WHEN** una línea queda disponible después de que su final ya haya pasado
- **THEN** el sistema la descarta y continúa con la siguiente línea vigente

### Requirement: El motor se detiene y se rearma al pausar
El sistema SHALL detener la actualización del reloj al pausarse la reproducción y SHALL rearmar el puntero de cues al reanudarse, de modo que las líneas del tramo pausado no se activen.

#### Scenario: Pausa
- **WHEN** el usuario pausa el vídeo
- **THEN** el reloj deja de actualizar y el puntero se rearmará al reanudar

#### Scenario: Reanudación
- **WHEN** el usuario reanuda la reproducción
- **THEN** el puntero se reposiciona a la línea vigente en el tiempo actual sin recitar las líneas perdidas

### Requirement: La pista adaptada se superpone al reproductor
El sistema SHALL renderizar la línea adaptada como un subtítulo superpuesto sobre el reproductor, con estilos propios que no alteran el layout ni los estilos de la página de YouTube.

#### Scenario: Contenedor de subtítulos
- **WHEN** hay una línea activa
- **THEN** el sistema muestra esa línea en un contenedor propio superpuesto al vídeo

#### Scenario: Línea no activa
- **WHEN** no hay ninguna línea activa en el tiempo actual
- **THEN** el contenedor de subtítulos no muestra texto

### Requirement: La UI se aísla en un shadow root
El sistema SHALL montar sus elementos de interfaz dentro de un shadow root, de modo que los estilos de la página de YouTube no alteren la UI de la extensión y los estilos de la extensión no alteren la página.

#### Scenario: Aislamiento de estilos
- **WHEN** la página de YouTube aplica reglas CSS globales a los elementos
- **THEN** la UI de la extensión mantiene su aspecto original

### Requirement: El overlay muestra el estado del pipeline
El sistema SHALL mostrar en el overlay el estado actual del proceso (extrayendo, adaptando, listo, error), el progreso en ventanas procesadas sobre ventanas totales y el idioma destino.

#### Scenario: Estado de extracción
- **WHEN** la extracción está en curso
- **THEN** el overlay muestra que se están extrayendo los subtítulos

#### Scenario: Estado de adaptación
- **WHEN** hay ventanas adaptándose
- **THEN** el overlay muestra el número de ventana en curso sobre el total

#### Scenario: Estado listo
- **WHEN** la primera ventana está lista
- **THEN** el overlay indica que la traducción está lista

#### Scenario: Estado de error
- **WHEN** una etapa del pipeline falla de forma irrecuperable
- **THEN** el overlay muestra el error y ofrece reintentar

### Requirement: El cambio de vídeo reinicia el pipeline
El sistema SHALL reaccionar a la navegación interna de YouTube a otro vídeo, sin recarga de página, cancelando el trabajo en curso y reiniciando la extracción y la adaptación para el nuevo vídeo.

#### Scenario: Navegación a otro vídeo
- **WHEN** el usuario navega a otro vídeo desde las recomendaciones o desde la búsqueda
- **THEN** el sistema descarta el estado del vídeo anterior y extrae los subtítulos del nuevo vídeo

#### Scenario: Los cues del vídeo anterior no se reutilizan
- **WHEN** el nuevo vídeo tiene un identificador distinto
- **THEN** el sistema no reutiliza los cues ni el guion cacheados del vídeo anterior

### Requirement: La extensión no modifica el audio ni el estado del reproductor
Mientras el cambio se limite a mostrar subtítulos, el sistema SHALL NOT alterar el volumen, el estado de silencio, la velocidad de reproducción ni ningún otro parámetro del reproductor, y SHALL NOT modificar ni subir ningún recurso del vídeo.

#### Scenario: Volumen intacto
- **WHEN** la extensión muestra la pista adaptada
- **THEN** el volumen del reproductor permanece en el valor que fijó el usuario

#### Scenario: Nada se sube a YouTube
- **WHEN** la extensión está activa
- **THEN** no se realiza ninguna petición de escritura al servicio de YouTube
