# cajeta-ifx-android — Spec

**Status:** Draft. Requirements for the **Android** backend of `ifx` (window / input / audio).
This is a requirements document (the *what* and *why*); the task breakdown is the plan.

## 1. Overview & why
`dev.cajeta.ifx.android` is the Android realization of the portable `ifx` contract
(`cajeta.ifx`). It FFI-binds Android system APIs and registers `WindowBackend` / `AudioBackend` /
`InputBackend` implementations through `BackendRegistry`. The app codes once against `ifx`; this
library is selected at runtime on Android (registry + probe + priority, with `CAJETA_IFX_*`
overrides). Umbrella design: the cajeta repo's `documents/cajeta-gfx/cajeta-gfx-spec.md` §9.

## 2. Capabilities (what it binds)
- **Window + Surface:** NDK ANativeWindow / GameActivity; produces the opaque `Surface` the gfx swapchain pairs with
  `VK_KHR_android_surface`.
- **Input:** AInputQueue / GameActivity (touch + gamepad) — keyboard/mouse arrive via the window event queue; gamepads via the input
  API.
- **Audio:** AAudio (+ OpenSL ES fallback) — output and capture.

> **Notes.** NDK/JNI; a single ANativeWindow surface; the Activity lifecycle drives the loop.

## 3. Goals
- Implement the full `ifx` SPI for Android behind the one portable API surface — no Android concept
  leaks to callers.
- Register + `probe()`/`priority()` so the dispatcher binds it on Android; honor `CAJETA_IFX_*`.
- Hand a correct opaque `Surface` to the gfx swapchain (`VK_KHR_android_surface`).
- Track Android API versioning **here**, isolated from the other backends.

## 4. Non-goals
- No portable-contract changes (those live in stdlib `ifx`).
- No graphics/render code — gfx owns the swapchain; this only supplies the `Surface`.
- No UI toolkit, input action-mapping, or audio mixing/DSP (engine / Glorias concerns).
- No codecs — capture/recording is the `cajeta-ifx-harness` backend + optional `VideoSink` providers.

## 5. References
- Umbrella design: `cajeta-gfx-spec.md` §9 (platform layer — §9.1 codegen-vs-HAL, §9.3 packaging,
  §9.4 binding model).
- Contract: `cajeta.ifx` — `Backend` / `WindowBackend` / `AudioBackend` / `InputBackend`.
- Selector: the `cajeta-ifx-backend` melt (target → backend).

---

## Appendix A — Vendor SDK details (2025-2026 research)

### Binding: C NDK + a JNI airlock
Rendering, input polling, and audio I/O are pure **NDK C** (direct FFI). But several capabilities
are Java-only and cross the **JNI boundary**: soft-keyboard text (GameTextInput), the `RECORD_AUDIO`
runtime permission, and lifecycle dialogs. → This backend ships a small **Java/Kotlin companion +
JNI glue** alongside the C code. The modern stack is the **Android Game Development Kit (AGDK)**:
GameActivity + GameTextInput + Paddleboat + Oboe + Swappy.

### SDK reference
| Domain | Modern (AGDK / NDK) | Legacy | C-ABI? | Min API |
|---|---|---|---|---|
| Window | **GameActivity** (SurfaceView, AppCompat-composable) → **`ANativeWindow`** (`<android/native_window.h>`); its `android_native_app_glue` fork → `android_main()`; surface on `APP_CMD_INIT_WINDOW`, gone on `APP_CMD_TERM_WINDOW` | NativeActivity | **C** | GameActivity → 19 |
| Surface | **`VK_KHR_android_surface`** (`vkCreateAndroidSurfaceKHR`, `ANativeWindow*`); **Swappy** frame pacing (Choreographer) | EGL/GLES | C | Vulkan 24 |
| Input | **`AInputEvent`** drained via `android_app_swap_input_buffers()`; **Paddleboat** (gamepad + rumble, hot-plug); **GameTextInput** (IME) | `AInputQueue`/`onInputEvent` | C (+JNI for IME text) | — |
| Audio | **AAudio** (`<aaudio/AAudio.h>`, API 26) / **Oboe** (C++ wrapper, recommended; AAudio on 27+, else OpenSL ES); low-latency = `AAUDIO_SHARING_MODE_EXCLUSIVE` + MMAP (OEM-dependent) | OpenSL ES (deprecated) | C/C++ (+JNI for `RECORD_AUDIO`) | AAudio 26 |

### Capability support & gaps (vs spec §9.7)
- **Single fullscreen `ANativeWindow`** — no multi-window/positioning/warp (n/a). Touch-first +
  optional gamepad (Paddleboat). Text via IME (GameTextInput), not raw keys.
- **Lifecycle-driven loop (no persistent `main()`):** the engine runs off the Activity lifecycle;
  **the surface can vanish mid-run** (`APP_CMD_TERM_WINDOW`) — GPU swapchain must be released/recreated.
  This is the canonical case behind `ifx`'s mandatory **surface-lost/recreated** events.
- **JNI airlock:** soft-keyboard text + `RECORD_AUDIO` permission prompt + lifecycle dialogs are
  Java → the companion handles them; `ifx`'s permission-state surface wraps the `RECORD_AUDIO` grant.
- **Fragmented OEM audio latency** — MMAP/EXCLUSIVE not guaranteed; query at runtime, degrade (Oboe's
  fallback chain). **Explicit frame pacing** (Swappy) is the engine's responsibility.
- **Floor: API 24 (Vulkan) / 26 (AAudio)**, GameActivity back to 19.

### References
- GameActivity: https://developer.android.com/games/agdk/game-activity · migrate: https://developer.android.com/games/agdk/game-activity/migrate-native-activity
- `vkCreateAndroidSurfaceKHR`: https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkCreateAndroidSurfaceKHR.html
- Paddleboat: https://developer.android.com/games/sdk/game-controller/controller · GameTextInput: https://developer.android.com/games/agdk/add-support-for-text-input
- AAudio: https://developer.android.com/ndk/guides/audio/aaudio/aaudio · Oboe: https://developer.android.com/games/sdk/oboe/low-latency-audio

---

## Appendix B — Interop mechanism (JNI is a C ABI)

**Feasibility: yes.** JNI is entirely C-ABI, and Cajeta `@Native` both calls and exports C symbols
(compiled through LLVM → a per-ABI NDK `.so`). No Java *language* support is required in Cajeta.

### Java → native: `RegisterNatives` in `JNI_OnLoad` (preferred)
Cajeta exports **`jint JNI_OnLoad(JavaVM*, void*)`** (one C symbol). Inside it: `GetEnv` → `FindClass`
the app classes → `RegisterNatives(clazz, JNINativeMethod[]{name, sig, fnptr})` binding
**Cajeta-compiled functions** to Java `native` methods. Preferred over the `Java_pkg_Class_method`
name-mangling path: export only `JNI_OnLoad`, hand ART raw function pointers, fail fast at load.
Do all `FindClass` here (correct class loader) and cache results as **global refs**.

### Native → Java: the `JNIEnv*` function-pointer table
`JNIEnv*` is a pointer to a C struct of ~230 function pointers — `FindClass`, `GetMethodID`/
`GetStaticMethodID`, `NewObject`, `Call*Method`, `Get/SetField`, `NewStringUTF` (modified UTF-8),
array ops. Every call is an ordinary indirect C call → all `@Native`-reachable. Method signatures
are JVM descriptors (`(Ljava/lang/String;I)Z`).

### Threading & references (absolute rules)
`JavaVM` is process-global; **`JNIEnv` is thread-local — never share across threads.** Native render/
audio threads: `GetEnv` → if `JNI_EDETACHED`, `AttachCurrentThread`; auto-detach via a
`pthread_key_create` TLS destructor. Manage refs: local refs on attached native threads are **not**
auto-freed (use `DeleteLocalRef`/`PushLocalFrame`); hold cross-call/thread handles as
`NewGlobalRef`/`DeleteGlobalRef`; compare with `IsSameObject`, never `==`.

### Build on the GameActivity glue
`android_native_app_glue` + GameActivity (compiled into the `.so` from AGDK source) give
`android_main(android_app*)` on a JVM-attached loop thread, already exposing the **`JavaVM`**,
**`JNIEnv`**, the **GameActivity `jobject`**, and the **`ANativeWindow`**. Hand-written JNI is needed
only for: **GameTextInput** (soft keyboard show/hide/state), the **`RECORD_AUDIO`** runtime-permission
prompt (`Activity.requestPermissions` on the `jobject` — no NDK permission API exists), and lifecycle
dialogs. These map to `ifx`'s permission state + lifecycle events.

### Packaging
A tiny **Java/Kotlin companion** is unavoidable for GameActivity: a `GameActivity` subclass +
`static { System.loadLibrary("cajeta_ifx_android"); }` + manifest `android.app.lib_name` + the
Activity declaration. Build: **Gradle/AGP** is the documented path (GameActivity via a **Prefab AAR**,
AGP 4.1+); the **no-Gradle minimum** is `aapt2 compile/link → javac/kotlinc + d8 → zip(+ lib/<abi>/
*.so) → zipalign → apksigner`. The cajeta build tool must drive one of these. Rust's
`android-activity` + `jni-rs` are the worked precedent for "non-Java language over JNI."

### Sources
- JNI tips (RegisterNatives, threads, refs): https://developer.android.com/training/articles/perf-jni
- GameActivity: https://developer.android.com/games/agdk/game-activity · GameTextInput: https://developer.android.com/games/agdk/game-activity/use-text-input
- JNI spec (functions): https://docs.oracle.com/en/java/javase/17/docs/specs/jni/functions.html
- android-activity (Rust precedent): https://github.com/rust-mobile/android-activity · jni-rs: https://docs.rs/jni
- No-Gradle APK: https://hereket.com/posts/android_from_command_line/
