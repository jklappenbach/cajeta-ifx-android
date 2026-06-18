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
