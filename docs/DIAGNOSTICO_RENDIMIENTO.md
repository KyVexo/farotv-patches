# Diagnóstico de sesiones y carga

## Prueba de sesión del 16 de noviembre de 2026

Se observaron seis minutos de FaroTV v1.7.0-dev.4 en el LX352 (Android 12, 1 GB de RAM, YouTube TV 8.02.300), con trece muestras cada treinta segundos. Se mantuvo el mismo proceso y no hubo errores de muestreo.

La PSS del proceso principal osciló entre 216761 y 239483 KiB. Tras salir del vídeo y verificar visualmente el inicio, las cinco últimas lecturas quedaron entre 223560 y 224124 KiB. En ese periodo, la mediana de CPU del proceso fue del 3,43 % de **un núcleo**, no del equipo completo. No se observó crecimiento continuo de memoria en esta prueba corta; no descarta problemas que aparezcan tras horas de uso.

Una traza de doce segundos capturó reproducción e interacción con el mando. Se leyeron los 239318 eventos declarados por la cabecera. El tiempo de CPU acumulado del proceso fue aproximadamente 6,55 segundos, repartido entre hilos que pueden ejecutarse en paralelo. Entre los mayores consumidores estaban el renderer de Chrome (2,18 s), su hilo GPU (1,30 s) y los compositores. También hubo trabajo de `kswapd0` del sistema (1,90 s). Esto **no identifica qué código JavaScript concreto consumió ese tiempo ni permite deducir la espera de red**. La captura añade coste de CPU, memoria y transferencia; no se usa como una comparación sin perturbaciones.

El muestreo se repite con:

```text
python tools/measure-tv-session.py --adb RUTA_AL_ADB --serial IP_DEL_TV:5555 --output CARPETA_DE_RESULTADOS --seconds 360 --interval 30
```

La herramienta solo observa. La reproducción y navegación se controlan aparte; `phase.txt` en la carpeta de resultados permite anotar la fase. Se debe verificar visualmente la pantalla: una etiqueta escrita por el operador no demuestra que se estuviera en el inicio. Los datos `/proc` de paginación y memoria disponible son del sistema entero, no solo de FaroTV.

## Parche opcional Performance diagnostics

Está **desactivado por defecto**. Al seleccionarlo al parchear el APK, habilita registros agregados bajo `TvPatches`:

- Cada quince segundos: cantidad y tamaño en caracteres de respuestas procesadas, tiempo de parseo, tiempo total de nuestros procesadores y desglose por función.
- Duraciones y espera hasta primer byte de recursos cuando el motor las expone. Solo agrupa categorías como `player`, `browse`, `next`, `img` y `script`; no registra las URL completas ni el contenido.
- Cantidad y duración de tareas largas si `PerformanceObserver` soporta `longtask`. Esas tareas pertenecen a toda la página, no necesariamente a FaroTV.
- En las consultas externas del puente nativo: host, espera de cola, duración de consulta, estado HTTP y longitud de la respuesta. Estas medidas solo cubren SponsorBlock y Return YouTube Dislike, no la descarga del vídeo de YouTube.

Si el motor no expone una API, el registro indica su ausencia; no se interpreta como duración cero. Las sumas de duración de recursos pueden solaparse y **no representan tiempo de carga total**. Los registros incluyen el coste de instrumentación.

El modo normal no crea estos observadores ni emite los resúmenes periódicos. Para terminar un diagnóstico se vuelve a parchear sin seleccionar `Performance diagnostics`, conservando la misma firma al instalar la actualización.

La primera publicación del parche opcional, dev.5, tenía un retorno de tipo incorrecto y fallaba al aplicarlo. Se corrigió en dev.6 y se aplicó e instaló sobre 8.02.300; no se instaló el APK fallido. Las pruebas automatizadas cubren desactivación por defecto, agregados sin contenido/URL, reinicio de ventanas y ausencia de observadores, además de las funciones anteriores. Es necesario probar también la aplicación real del parche sobre el APK.

## Datos recogidos con dev.6

Se obtuvieron cinco ventanas de diagnóstico. El motor expuso observadores de recursos y tareas largas.

- En la primera ventana, nuestros procesadores de respuestas consumieron 66,5 ms acumulados para 54 respuestas; la ejecución más larga de esos procesadores fue 24,4 ms. El parseo contabilizado sumó 44,3 ms.
- La página registró 36 tareas largas que sumaron 8559 ms, con una de 1238 ms. Estas tareas incluyen código e inicialización de la página completa. El tiempo de nuestros procesadores no incluye todo el arranque de los módulos, por lo que no se resta de esa cifra para atribuir el resto a un componente concreto.
- Una petición `browse` tardó 1258,8 ms, de los cuales 1204,9 ms transcurrieron hasta el primer byte. Otra ventana registró `next` en 844,2 ms, con 806,5 ms hasta el primer byte. Son observaciones puntuales de la ruta de red/servidor, no una garantía ni un promedio de YouTube.
- Una consulta asíncrona de SponsorBlock esperó 4 ms en cola y 1385 ms en la consulta, con HTTP 404: no había segmentos disponibles. No se interpreta esa duración como bloqueo de la reproducción.
- En otra ventana el máximo de nuestros procesadores llegó a 34,7 ms. Reducir su trabajo sigue siendo útil, aunque estos datos no justifican prometer que eliminará todas las pausas.

## Mejora de iconos en dev.7

`icons.js` reconstruía ilustraciones y adoptaba otra hoja CSS cada vez que se volvía a inyectar el paquete en el mismo documento. Ahora reutiliza su hoja si sigue presente. Si la hoja desaparece o la carga falló, permite reconstruirla.

La prueba de cien reinyecciones compara **cien hojas/escrituras CSS antes y una después**. No equivale a afirmar que un TV real acumulara cien copias durante los seis minutos de observación. El diagnóstico informa también del número de hojas de iconos para comprobarlo en el motor real. Las diecinueve pruebas incluyen esa reutilización y la recuperación tras un fallo.

```text
node tools/benchmark-runtime.cjs v1.7.0-dev.6
```

Esta modificación evita trabajo y copias propias de FaroTV. La espera de los recursos remotos y las tareas de inicialización del resto de la página siguen requiriendo un análisis separado.
