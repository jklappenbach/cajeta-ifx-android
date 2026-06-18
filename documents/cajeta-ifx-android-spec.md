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
