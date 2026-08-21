# Voz neural Piper en Windows para Read Selection

**Fecha:** 2026-08-21
**Rama:** `feature/piper-tts-windows`
**Relacionado:** `fix/lectura-voz-windows` (fix previo que conectó `read_selection` a `speechSynthesis` como respaldo universal)

## Contexto y problema

Escriba ya tiene un motor de voz neuronal 100% local (`managers/tts.rs`): runtime `sherpa-onnx` + voz Piper `es_MX-claude-high`, descargado bajo demanda con verificación SHA256, igual patrón que los modelos Whisper de transcripción. Ese motor alimenta una cascada de 3 pasos ya usada por "Sesiones" y el "Traductor" (`commands/conversation.rs::speak_native`):

1. Voz neural Piper incluida (si está instalada y el idioma es español)
2. Voz nativa del sistema operativo (`say` en macOS)
3. `false` → el frontend cae a `speechSynthesis` del navegador

**Problema 1 — plataforma:** `managers/tts.rs` está compilado con `#[cfg(all(target_os = "macos", target_arch = "aarch64"))]`. En cualquier otra plataforma (incluida Windows, donde vive Alex) la función `setup_lang` devuelve directamente `Err("La voz neural v1 es solo para macOS Apple Silicon")`, y toda la cascada cae siempre al escalón 3.

**Problema 2 — desconexión de `read_selection`:** aunque el runtime existiera en Windows, `ReadSelectionAction::start` (`actions.rs:1730`) llama a `speak_native(&ah, &text, "system")` con el `engine` fijo en `"system"`. El propio `speak_native` salta el motor neural cuando `engine == "system"` (`conversation.rs:266`, comentario: "salvo que el usuario prefiera la del sistema"). Es decir: **aunque el runtime de Windows exista, `read_selection` seguiría sin usarlo** hasta cambiar ese valor.

Ambos problemas hay que resolverlos para que `alt+shift+r` suene con la voz Piper en vez de la voz robótica de `speechSynthesis` (`es-MX Raul`/`Sabina`, únicas voces locales que expone Windows/Edge).

## Objetivo

`alt+shift+r` (Read Selection) lee la selección con la voz neural Piper en Windows, con la misma cascada de respaldo que ya usan Sesiones y el Traductor. Sesiones y Traductor quedan cubiertos automáticamente porque ya llaman a la cascada compartida — no requieren cambios propios.

## No-objetivos (fuera de alcance de esta iteración)

- No se toca el comportamiento en macOS (sigue igual).
- No se agregan más idiomas ni más voces al catálogo (`VOICES` en `tts.rs` mantiene solo `es`/`en` como hoy).
- No se construye un selector de voz en la UI — sigue sin haber uno; esto solo mejora qué voz suena por debajo.
- No se resuelve el bug intermitente de `read_selection` documentado en la memoria del proyecto (`alt+shift+r` sin reaccionar una vez, sin causa confirmada) — es un problema aparte.
- No se toca el auto-updater roto (apunta a `AlejandroAP9/Escriba`, repo equivocado) — anotado, no relacionado.

## Diseño técnico

### 1. Runtime Windows en `managers/tts.rs`

Se agrega una rama de compilación para Windows x64, en paralelo a la de macOS (no se toca la existente):

- **Pin de runtime:** `sherpa-onnx-v1.13.4-win-x64-shared-MD-Release.tar.bz2`, del mismo release `v1.13.4` ya pinneado para macOS (confirmado que existe oficialmente en `k2-fsa/sherpa-onnx`). Elegido `shared` (equivalente al `shared` de macOS, no `static`), `MD` (runtime dinámico de MSVC, el estándar en binarios distribuidos), `Release` (sin símbolos de debug), sin sufijo `-lib` (ese variant excluye el ejecutable CLI) ni `no-tts` (excluiría justo lo que necesitamos).
- **Pendiente de implementación (no adivinar):** descargar ese asset una vez, calcular su SHA256 real y su tamaño exacto en bytes, y agregarlos como constantes — mismo patrón que `RUNTIME_SHA256`/`RUNTIME_SIZE` de macOS. **Nunca aproximar el hash**: `download_verified` aborta si no calza, así que un hash inventado deja la voz permanentemente rota en Windows.
- **Voz:** se reutiliza sin cambios el mismo tarball `vits-piper-es_MX-claude-high.tar.bz2` y su SHA256 ya pinneado — la voz es multiplataforma, solo cambia el runtime que la ejecuta.
- **Extracción:** `tar.exe` nativo de Windows (bsdtar/libarchive, incluido de fábrica desde Windows 10 1803+ y en Windows 11) en vez de `/usr/bin/tar`. Mismo patrón que macOS, sin dependencias nuevas. Riesgo residual: si algún Windows del cliente no tiene `tar.exe` en PATH (versiones muy viejas o "N"/LTSC recortadas), la extracción falla con error explícito — no hay fallback silencioso.
- **Re-firma (`resign_runtime`):** no aplica en Windows. Se implementa como no-op bajo `#[cfg(windows)]` (Windows no exige re-firma para ejecutar binarios descargados).
- **Rutas de sistema:** reemplazar los `Command::new("/usr/bin/...")` por sus equivalentes sin ruta absoluta en Windows (`tar` resuelto por PATH).

### 2. Reproducción de audio

macOS reproduce el WAV generado invocando `/usr/bin/afplay` como proceso hijo (`Child`), guardado en `PLAYING: Mutex<Option<Child>>` para poder cortarlo.

Windows no tiene un equivalente directo tan simple por línea de comandos. En vez de improvisar uno (p. ej. PowerShell `System.Media.SoundPlayer`, que arrastra ~300-500ms de arranque de proceso por cada lectura y es frágil con rutas/comillas), se usa **`rodio`** — ya es dependencia del proyecto (`Cargo.toml:71`, mismo fork `cjpais/rodio` que usa el Intérprete para su sink de audio).

Esto cambia el tipo interno de reproducción en Windows (un `Sink` de rodio en vez de un `Child` de proceso), pero **la interfaz pública no cambia**: `speak_blocking`, `is_playing`, `stop` mantienen la misma firma. `PLAYING` pasa a ser condicional por plataforma (`Child` en macOS, `rodio::Sink` en Windows) o se abstrae con un enum pequeño si conviene mantener un solo tipo — decisión de implementación, no de spec.

### 3. Conectar `read_selection` a la cascada completa

En `actions.rs:1730`, cambiar:

```rust
if !crate::commands::conversation::speak_native(&ah, &text, "system").await {
```

a pasar un `engine` que NO sea `"system"` (p. ej. `""`), para que `speak_native` intente el motor neural Piper primero, igual que Sesiones. Esta fue una decisión explícita de Alex (confirmada en la sesión de diseño): `read_selection` debe sonar con Piper cuando esté disponible, no quedar forzado a la voz del sistema.

### 4. Instalación / UI

**Sin cambios en frontend.** El botón de instalación de la voz neural ya existe en Ajustes de Sesiones (`ConversationSettings.tsx:597-654`, `commands.ttsStatus()` / `commands.ttsSetup()`, con barra de progreso vía el evento `tts-setup-progress`). Esos comandos Tauri ya son genéricos por plataforma vía `#[cfg]` interno de `tts.rs` — en cuanto el runtime de Windows compile, ese mismo botón descarga (~95 MB: runtime + voz) y deja lista la voz para los tres consumidores (Sesiones, Traductor, Read Selection).

## Manejo de errores

Se mantiene el mismo patrón defensivo que ya existe, sin inventar uno nuevo:

- **Descarga corrupta o interrumpida:** `download_verified` ya borra el archivo temporal si el SHA256 no calza — se hereda sin cambios.
- **Extracción falla (`tar.exe` ausente o corrupta):** error explícito propagado al frontend vía el `Result` de `tts_setup`; no se instala nada a medias.
- **Runtime no instalado o síntesis falla en tiempo de lectura:** `speak_native` ya cae sola al escalón 2/3 de la cascada — el usuario nunca se queda mudo, en el peor caso vuelve a oír `speechSynthesis` (el comportamiento de hoy).
- **`tar.exe` no encontrado en PATH:** no hay fallback de extracción alternativo en esta iteración (ver riesgo residual arriba) — se reporta como error de instalación, no como fallo silencioso.

## Plan de verificación

No hay suite de tests automatizados para esta capa (dependencia de un binario externo real + hardware de audio). Verificación manual:

1. Compilar para Windows x64 y confirmar que `tts_setup` descarga, verifica el hash y extrae sin error.
2. Desde Ajustes de Sesiones, instalar la voz neural y confirmar que `ttsStatus()` pasa a `true`.
3. Seleccionar texto en cualquier ventana y presionar `alt+shift+r`: confirmar que suena la voz Piper (no la voz robótica de `speechSynthesis`) y que el toggle (segunda pulsación) corta la lectura igual que antes.
4. Probar con la voz neural **no instalada** (entorno limpio): confirmar que cae a `speechSynthesis` sin errores ni cuelgues — la cascada de respaldo sigue intacta.
5. Repetir el mismo flujo en Sesiones y en el Traductor para confirmar que también se benefician sin cambios propios.

## Riesgos conocidos

- El binario compilado para Windows en GitHub Actions del fork es el único método de build disponible hoy (Smart App Control bloquea `cargo` localmente en la máquina de Alex — ver memoria del proyecto). Cualquier iteración de este cambio pasa por ese mismo pipeline.
- El SHA256 y tamaño del asset de Windows deben calcularse una vez, a mano, antes de poder compilar — no son inventables ni aproximables.
- Cambiar el `engine` de `read_selection` de `"system"` a `""` es un cambio de comportamiento que también aplicaría en macOS (Read Selection ahí empezaría a preferir Piper sobre la voz del sistema). No se identificó como un problema en la sesión de diseño, pero vale mencionarlo explícitamente como efecto colateral esperado, no accidental.
