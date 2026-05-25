# TECHNICAL_REPOSITORY_REPORT

## 1) Project Overview
`ScreenRecorder` es una app Android nativa para grabación de pantalla basada en `MediaProjection`, orientada a flujo rápido de inicio/parada y gestión local de grabaciones. Incluye:
- Inicio/parada de grabación desde pantalla principal (`MainActivity`) y desde Quick Settings Tile (`ScreenRecordTileService`).
- Controles flotantes durante grabación (pausar/reanudar/detener) vía `RecordingOverlayService`.
- Grabación con opciones de micrófono y audio interno (según versión Android y permisos).
- Listado local de videos guardados y acciones de renombrar/compartir/borrar.
- Hoja de ajustes con calidad, tema, colores dinámicos y “show touches”.

## 2) Repository Summary
- **Tipo de repo:** app Android monomódulo (`:app`).
- **Lenguaje principal:** Kotlin (UI en XML + ViewBinding, no Compose).
- **Stack Android:** AppCompat + Material 3 (Expressive), RecyclerView, MediaProjection, MediaCodec/MediaMuxer, MediaStore.
- **Gradle:** Kotlin DSL (`*.gradle.kts`), AGP 8.13.2, Kotlin 2.0.21, wrapper Gradle 8.13.
- **Módulos:** solo `app`.
- **Paquete raíz:** `com.haseeb.recorder`.
- **Requisitos de build declarados/código:** Java 17, `compileSdk 36`, `minSdk 26`, `targetSdk 36`.

## 3) High-Level Architecture
- **Módulo `app`:** concentra UI, servicios, configuración, capa de grabación y acceso a MediaStore.
- **Estructura Kotlin:** clases principales bajo `app/src/main/java/com/haseeb/recorder/`.
- **Capa UI:** `MainActivity`, `AboutActivity`, `SettingsBottomSheet`, `VideoAdapter`, layouts XML.
- **Servicios:**
  - `ScreenRecordService`: pipeline de captura/encode/mux.
  - `RecordingOverlayService`: controles flotantes.
  - `ScreenRecordTileService`: entrada desde Tile QS.
- **MediaProjection/recording layer:** se solicita token en `MediaProjectionPermissionActivity` y se pasa al servicio.
- **Foreground service:** `ScreenRecordService` se eleva a FGS con tipo `mediaProjection|microphone` según caso.
- **Permisos:** runtime + especiales (`SYSTEM_ALERT_WINDOW`, `WRITE_SETTINGS`) + FGS + audio + media read.
- **Storage/output:** `MediaStore` (Q+) con `RELATIVE_PATH=DCIM/ScreenRecorder`; pre-Q usa ruta pública DCIM.
- **QS Tile:** implementado.
- **Overlay flotante:** implementado.

## 4) Repository Tree Map
```text
.
├─ settings.gradle.kts                # Incluye módulo :app
├─ build.gradle.kts                   # Plugins raíz (AGP/Kotlin)
├─ gradle/wrapper/gradle-wrapper.properties
├─ app/
│  ├─ build.gradle.kts                # Config Android/deps
│  ├─ proguard-rules.pro
│  └─ src/main/
│     ├─ AndroidManifest.xml          # Componentes, permisos, FGS types
│     ├─ java/com/haseeb/recorder/
│     │  ├─ MainActivity.kt           # Pantalla principal, permisos, listado, trigger de grabación
│     │  ├─ MediaProjectionPermissionActivity.kt # Solicita permiso de captura y lanza servicio
│     │  ├─ ScreenRecordService.kt    # Núcleo de grabación (video/audio/mux)
│     │  ├─ RecordingOverlayService.kt# Overlay flotante pausar/reanudar/detener
│     │  ├─ ScreenRecordTileService.kt# Inicio/parada desde Quick Settings Tile
│     │  ├─ ConfigManager.kt          # Persistencia de opciones y cálculo de calidad/bitrate
│     │  ├─ SettingsBottomSheet.kt    # UI de configuración
│     │  ├─ VideoAdapter.kt           # Lista de videos + share/rename/delete
│     │  ├─ AboutActivity.kt          # Pantalla about
│     │  ├─ HaseebApplication.kt      # Inicialización tema/dynamic colors
│     │  ├─ EdgeToEdge.kt             # Helpers de insets
│     │  └─ VideoFile.kt              # Modelo de item de video
│     └─ res/layout/                  # activity_main, settings sheet, overlay, countdown, etc.
└─ fastlane/metadata/android/en-US/   # Metadatos store (texto/changelogs/screenshots)
```

## 5) Android Components
### Activities
- `MainActivity` (`exported=true`, launcher).
- `AboutActivity` (`exported=false`).
- `MediaProjectionPermissionActivity` (`exported=false`, `excludeFromRecents=true`, tema transparente).

### Services
- `ScreenRecordService` (`exported=false`, `foregroundServiceType="mediaProjection|microphone"`).
- `RecordingOverlayService` (`exported=false`).
- `ScreenRecordTileService` (`exported=true`, permiso `BIND_QUICK_SETTINGS_TILE`, `intent-filter` QS tile).

### Broadcast receivers
- No receiver declarado en Manifest. Se usan receivers dinámicos in-app.

### Permisos declarados
- `INTERNET`.
- `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_MEDIA_PROJECTION`, `FOREGROUND_SERVICE_MICROPHONE`.
- `WRITE_SETTINGS`, `SYSTEM_ALERT_WINDOW`.
- `RECORD_AUDIO`, `POST_NOTIFICATIONS`.
- `READ_MEDIA_VIDEO`.
- `READ_EXTERNAL_STORAGE` (maxSdk 32), `WRITE_EXTERNAL_STORAGE` (maxSdk 28).

### Compatibilidad/versionado
- `minSdk 26` limita ejecución a Android 8+.
- Audio interno depende de Q+ (`AudioPlaybackCapture`).
- Manejo de permisos de media cambia por versión (T+ usa `READ_MEDIA_VIDEO`).

## 6) Recording Pipeline
1. **Disparo:** desde FAB (`MainActivity.handleRecordAction`) o tile.
2. **Permiso MediaProjection:** `MediaProjectionPermissionActivity` solicita consentimiento del sistema.
3. **Start service:** token (`resultCode + data Intent`) enviado por `ACTION_START` a `ScreenRecordService`.
4. **FGS + overlay:** servicio entra en foreground (tipos dinámicos) y levanta `RecordingOverlayService`.
5. **Inicialización salida:**
   - Q+: inserta registro en `MediaStore.Video` con nombre `ScreenRecord_yyyyMMdd_HHmmss.mp4`.
   - pre-Q: crea archivo en `DCIM/ScreenRecorder`.
6. **Video encode:** `MediaCodec` (HEVC si habilitado y soportado, si no AVC), surface input, VBR, bitrate calculado por `ConfigManager`, FPS configurable.
7. **Audio encode (opcional):** AAC mono 48kHz, mezcla de mic + audio interno cuando ambos están activos.
8. **VirtualDisplay:** `MediaProjection.createVirtualDisplay(...)` con resolución escalada y densidad ajustada.
9. **Drenado de encoders:** hilos dedicados audio/video -> `MediaMuxer` MP4.
10. **Pause/Resume:** flags `isPaused`, ajuste de PTS descontando duración pausada para continuidad temporal.
11. **Stop/Cleanup:** detener hilos/encoders/records/projection/muxer, restaurar show touches, notificar estado.
12. **Errores:** `try/catch` en inicio/escritura; en fallo limpia recursos y detiene servicio.

## 7) Audio Handling
- **Micrófono:** vía `AudioRecord(MediaRecorder.AudioSource.MIC)` cuando setting + permiso `RECORD_AUDIO`.
- **Audio interno:** vía `AudioPlaybackCaptureConfiguration` en Android Q+.
- **Mezcla:** suma sample a sample (mono) con clamping `Short.MIN..MAX`.
- **Restricciones:**
  - Interno no disponible < Q.
  - Depende de políticas de captura de otras apps (no todo audio de terceros es capturable).
- **Permisos:** `RECORD_AUDIO` requerido para flujo de audio configurable por app.
- **Riesgo:** mezcla simple puede introducir clipping/distorsión en señales altas.

## 8) Floating Controls / Pause / Resume / Stop
- **Implementación:** `RecordingOverlayService` + layout `layout_recording_overlay.xml`.
- **Visualización:** overlay de sistema (requiere `SYSTEM_ALERT_WINDOW`), con timer y botones pause/stop.
- **Interacción:** botones envían acciones `ACTION_PAUSE`, `ACTION_RESUME`, `ACTION_STOP` a `ScreenRecordService`.
- **¿Puede salir en el video?:** sí, por diseño de overlay sobre pantalla capturada puede aparecer en la grabación.
- **Para ocultarlo en grabación (dirección técnica):**
  - revisar tipo de ventana/flags en `RecordingOverlayService`; migrar a canal de control no dibujado sobre display capturado (p.ej. notificación + tile + activity transient).
  - opcionalmente detectar/display target y excluir capa si API/dispositivo lo permite (limitado por plataforma).
- **Para control pause/play transparente:** cambiar `layout_recording_overlay.xml` (alpha, fondo, hit area) y lógica de interacción/animación en `RecordingOverlayService`.

## 9) UI and UX Structure
- **Tecnología UI:** XML + ViewBinding + Material Components; no Compose.
- **Pantallas:**
  - `MainActivity`: lista de videos + FAB grabar/parar + botón settings.
  - `SettingsBottomSheet`: audio, calidad, apariencia, advanced, about.
  - `AboutActivity`: lista de enlaces/maintainers.
  - `MediaProjectionPermissionActivity`: flujo técnico de permiso/cuenta regresiva.
- **Theme/Material 3:** `Theme.Material3Expressive.DynamicColors.DayNight.NoActionBar` + colores dinámicos opcionales.
- **Countdown:** layout `activity_countdown.xml` y lógica en `MediaProjectionPermissionActivity`.
- **Interacciones clave:** grabar/parar, toggle mic/system audio/show touches, renombrar/compartir/borrar videos.

## 10) Build System
- `settings.gradle.kts`: repos `google/mavenCentral/gradlePluginPortal`, include `:app`.
- `build.gradle.kts` raíz:
  - `com.android.application` 8.13.2
  - `org.jetbrains.kotlin.android` 2.0.21
- `app/build.gradle.kts`:
  - `compileSdk=36`, `minSdk=26`, `targetSdk=36`
  - `versionCode=6`, `versionName=4.0`
  - Java/Kotlin target 17
  - `viewBinding=true`
  - release sin minify
- Wrapper: Gradle 8.13 (`gradle-wrapper.properties`).
- Dependencias clave: appcompat, material, recycler, constraintlayout, glide.
- Signing/release: no configuración custom de signing visible en `build.gradle.kts`; existe archivo `keystore` en raíz (revisar manejo seguro fuera de repo).
- Fastlane metadata: sí, presente en `fastlane/metadata/android/en-US`.

## 11) Data and Storage
- **Destino grabaciones:** `DCIM/ScreenRecorder`.
- **Q+ (scoped storage):** inserta en MediaStore con `RELATIVE_PATH`; escritura mediante `ParcelFileDescriptor`.
- **Pre-Q:** ruta directa pública DCIM (legacy external storage).
- **Naming:** prefijo `ScreenRecord_` + timestamp `yyyyMMdd_HHmmss` + `.mp4`.
- **Listado UI:** query MediaStore filtrando por `DISPLAY_NAME LIKE 'ScreenRecord_%.mp4'`.

## 12) Security and Privacy Review
- **Captura de pantalla:** alto impacto de privacidad (posible captura de datos sensibles visibles).
- **Captura de audio:** mic + interno incrementan sensibilidad legal/privacidad.
- **Overlay:** riesgo UX/seguridad (draw-over-other-apps), además puede exponerse en video.
- **Exported components:** solo `MainActivity` launcher y `ScreenRecordTileService` (requerido para QS).
- **Permisos amplios:** incluye `WRITE_SETTINGS` y overlay, ambos sensibles para revisión store.
- **FGS notification:** presente (requisito de transparencia al usuario).
- **Play Store concerns potenciales:** justificar claramente uso de MediaProjection, RECORD_AUDIO, overlay y foreground service types.

## 13) Performance Review
- **Bitrate/resolución:** calculados por `ConfigManager`; riesgo de sobrecarga en dispositivos modestos en calidades altas.
- **Encode pipeline:** hilos separados para audio/video; correcto para evitar bloqueo UI.
- **Mezcla audio:** CPU lineal sobre frames; generalmente manejable pero dependiente de sample rate constante.
- **Memoria:** buffers manuales y codec buffers; riesgo moderado si múltiples recursos no se liberan en edge cases.
- **Pause/resume:** ajuste PTS reduce desincronización, pero complejidad temporal puede fallar en devices específicos.
- **FGS estabilidad:** `START_NOT_STICKY`; si sistema mata servicio no hay reanudación automática.

## 14) Error Handling and Edge Cases
Estado observado:
- **Denegación permisos runtime:** flujo retorna y no continúa.
- **Denegación MediaProjection:** activity reporta/termina.
- **Service killed:** no recuperación automática (`START_NOT_STICKY`).
- **Rotación pantalla:** no se identifica lógica explícita de reconfiguración de VirtualDisplay durante grabación.
- **Espacio insuficiente:** no se observa verificación preventiva explícita; dependería de fallos del muxer/escritura.
- **Audio permiso denegado:** grabación puede iniciar sin audio según checks.
- **Diferencias Android:** sí hay ramas por SDK para permisos/storage/audio interno.
- **Background/lockscreen:** posible continuidad por FGS, pero sin matriz de comportamiento formal en repo.
- **Interrupciones (llamadas/otros):** no hay manejo dedicado visible más allá de callbacks de projection/stop.

## 15) Testing and Verification (manual)
Checklist práctico:
1. Build debug APK (`./gradlew :app:assembleDebug`).
2. Instalar APK y abrir app.
3. Validar flujo de permisos (audio, notificaciones, overlay, write settings).
4. Iniciar grabación desde app.
5. Iniciar/parar desde Quick Settings tile.
6. Probar pause/resume/stop desde overlay.
7. Probar mic ON/OFF y audio interno ON/OFF (Q+).
8. Rotar pantalla durante grabación.
9. Bloquear pantalla/enviar app a background.
10. Verificar archivo final (duración, reproducción, A/V sync, metadata).
11. Validar eliminación/renombrado/compartido desde lista.
12. Ejecutar en Android 8–16 (al menos un dispositivo por rango API relevante).

## 16) Current Technical Risks (priorizado)
### High
1. **Overlay visible en grabación / UX-política**
   - Archivos: `RecordingOverlayService.kt`, `layout_recording_overlay.xml`, `ScreenRecordService.kt`.
   - Impacto: contaminación visual del output + posible fricción de cumplimiento/política UX.
   - Dirección: canal de control alterno (notif/tile) o rediseño overlay no intrusivo.

2. **Gestión de errores de almacenamiento limitada**
   - Archivos: `ScreenRecordService.kt`.
   - Impacto: grabaciones corruptas/cero bytes en condiciones límite.
   - Dirección: pre-check de espacio, validación de URI/PFD, notificación explícita por causa.

### Medium
1. **Manejo lifecycle/kill del servicio**
   - Archivos: `ScreenRecordService.kt`.
   - Impacto: interrupción sin recuperación en memoria baja.
   - Dirección: estrategia resiliente (estado persistido + restart controlado si aplica).

2. **Compatibilidad audio interno heterogénea**
   - Archivos: `ScreenRecordService.kt`, `SettingsBottomSheet.kt`.
   - Impacto: expectativas de usuario no cumplidas en ciertos apps/devices.
   - Dirección: UI con aviso contextual de limitaciones por versión/política de fuente.

### Low
1. **Dependencias alpha/beta en UI stack**
   - Archivo: `app/build.gradle.kts`.
   - Impacto: regresiones menores de compatibilidad visual.
   - Dirección: fijar a versiones estables en release pipeline.

## 17) Suggested Roadmap
### Easy
- Mensajes de error más accionables (permiso, storage, fallo encoder).
- Avisos UI para limitaciones de audio interno por versión.
- Mejor feedback tras stop (abrir carpeta/video recién generado).

### Medium
- Máquina de estados explícita de grabación (Idle/Starting/Recording/Paused/Stopping/Error).
- Validación previa de espacio libre y health checks de muxer.
- Hardening de overlay (posición persistente, fallback si permiso revocado).

### Advanced
- Estrategia para ocultar controles en output (control remoto por notificación/tile + overlay mínimo opcional).
- Control transparente pause/play configurable (alpha/shape/hit slop/auto-hide inteligente).
- CI/release automation (build matrix, lint, static checks, generación changelog/metadata fastlane).
- Preparación para rename de paquete/app: inventario centralizado de IDs, authority y branding assets.

## 18) Modification Guide for Future Codex Sessions
1. **Primero inspeccionar:**
   - `AndroidManifest.xml`
   - `ScreenRecordService.kt`
   - `RecordingOverlayService.kt`
   - `MediaProjectionPermissionActivity.kt`
   - `ConfigManager.kt`
   - `MainActivity.kt`
2. **Archivos de alto riesgo:** `ScreenRecordService.kt` (pipeline crítico), `AndroidManifest.xml` (permisos/componentes), `RecordingOverlayService.kt` (UX/permissions).
3. **Lógica de grabación:** `ScreenRecordService.kt`.
4. **Lógica de UI:** `MainActivity.kt`, `SettingsBottomSheet.kt`, layouts XML.
5. **Cambios de permisos/manifest:** solo en `app/src/main/AndroidManifest.xml` + validación de flujo runtime en `MainActivity`.
6. **Cambios pequeños seguros:** strings, textos UI, estilos no funcionales, documentación.
7. **Evitar tocar sin necesidad:** codec/audio timing, muxer, manejo de PTS, tipos de FGS, overlay window flags.

## 19) Exact File Reference Index
| File path | Propósito | Importancia | Notas |
|---|---|---:|---|
| `settings.gradle.kts` | Definición de módulos/repositorios | Alta | Monomódulo `:app` |
| `build.gradle.kts` | Versiones plugins AGP/Kotlin | Alta | Controla toolchain global |
| `gradle/wrapper/gradle-wrapper.properties` | Versión Gradle wrapper | Alta | `8.13` |
| `app/build.gradle.kts` | Config Android/dependencias | Crítica | SDKs, versiones, build types |
| `app/src/main/AndroidManifest.xml` | Permisos y componentes Android | Crítica | FGS types + tile + overlay |
| `app/src/main/java/com/haseeb/recorder/ScreenRecordService.kt` | Pipeline grabación A/V | Crítica | Núcleo funcional |
| `app/src/main/java/com/haseeb/recorder/MediaProjectionPermissionActivity.kt` | Solicitud permiso MediaProjection | Crítica | Entrada al servicio |
| `app/src/main/java/com/haseeb/recorder/RecordingOverlayService.kt` | Overlay de control runtime | Alta | Pause/Resume/Stop |
| `app/src/main/java/com/haseeb/recorder/ScreenRecordTileService.kt` | Integración Quick Settings | Alta | Trigger rápido externo |
| `app/src/main/java/com/haseeb/recorder/MainActivity.kt` | UI principal + permisos + listado | Alta | Orquestación de usuario |
| `app/src/main/java/com/haseeb/recorder/ConfigManager.kt` | Persistencia settings y calidad | Alta | Impacta bitrate/resolución |
| `app/src/main/java/com/haseeb/recorder/SettingsBottomSheet.kt` | Ajustes usuario | Media | Cambios de config runtime |
| `app/src/main/java/com/haseeb/recorder/VideoAdapter.kt` | Gestión de lista/acciones video | Media | Share/rename/delete |
| `app/src/main/res/layout/layout_recording_overlay.xml` | UI de controles flotantes | Alta | Visibilidad en captura |
| `app/src/main/res/layout/activity_main.xml` | UI principal | Media | Lista + FAB + appbar |
| `app/src/main/res/layout/layout_settings_sheet.xml` | UI ajustes | Media | Toggles y botones calidad |
| `app/src/main/res/layout/activity_countdown.xml` | UI countdown | Media | Pre-start visual |
| `app/src/main/res/values/strings.xml` | Textos UX/notificaciones | Media | Mensajería y labels tile |
| `fastlane/metadata/android/en-US/*` | Metadatos de publicación | Media | Proceso release/store |

## 20) Open Questions
1. No se verificó ejecución real en dispositivo/emulador en este cambio documental; comportamiento exacto multi-OEM no confirmado.
2. No se auditó exhaustivamente la implementación completa de `RecordingOverlayService.kt`/`MediaProjectionPermissionActivity.kt` más allá de secciones necesarias; posibles detalles finos de animación/flags no documentados línea a línea.
3. No hay evidencia en este análisis de pipeline CI automatizado (GitHub Actions, etc.); podría existir fuera de archivos inspeccionados.
4. La presencia del archivo `keystore` en raíz requiere decisión de seguridad del equipo (origen/rotación/exclusión), no inferible solo con lectura funcional.
