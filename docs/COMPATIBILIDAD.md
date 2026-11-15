# Compatibilidad de FaroTV

## Estado del 15 de noviembre de 2026

Los puntos de cierre en segundo plano y prueba desde la interfaz de Morphe Manager quedan pendientes por decisión del usuario. Esta rama solo trabaja en compatibilidad, a partir de la estable v1.4.2; no incluye el commit local 714c164.

| YouTube TV | ABI examinada | Firma original | Parcheo normal | Con Spoof device | Prueba en TV |
| --- | --- | --- | --- | --- | --- |
| 7.32.300 | armeabi-v7a | Google | Validado previamente | Validado previamente | LX352 y OTT-01 |
| 8.02.300 | armeabi-v7a | Google, verificada | Correcto con v1.4.2 y --validate | Correcto con v1.4.2 y --validate | Pendiente; experimental |

APK de 8.02.300: versión de paquete 802300320, 75 163 552 bytes. SHA-256 del archivo:
`a4fab76d43098e62fa2fcb3a20df0693475d4c2e4872bc13477e619b1d33b8cb`.

Fuente: https://www.apkmirror.com/apk/google-inc/youtube-for-android-tv-android-tv/youtube-for-android-tv-8-02-300-release/

Un parcheo correcto no demuestra reproducción, autenticación ni funcionamiento de los ajustes en tiempo de ejecución. La versión 8.02.300 se declara experimental hasta comprobar esos recorridos.

La compilación local con la declaración experimental terminó correctamente. Se repitieron los cuatro casos (7.32.300 y 8.02.300, con y sin Spoof device) sin `--force`: todos pasaron `--validate`, reconstrucción del APK y comprobación del nombre de paquete. Los resultados JSON no contienen parches fallidos.

## Repetir la comprobación

`tools/Test-Compatibility.ps1` recibe un APK original, el bundle, morphe-desktop, la carpeta build-tools del SDK y una carpeta de salida. Verifica el certificado de Google, parchea las variantes normal y spoof con resolución única de fingerprints y comprueba el paquete de salida. No instala las aplicaciones. `-ForceVersion` sirve únicamente para explorar un APK aún no declarado compatible.

## Arquitecturas

La prueba actual cubre ARM de 32 bits. El editor ELF y la biblioteca libytvshim.so de Spoof device solo cubren esa arquitectura. ARM64 necesita su biblioteca, un editor ELF64 y un APK original con el que validar las importaciones. No anunciar ARM64 ni x86 como compatibles a partir de estas pruebas.
