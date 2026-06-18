# cajeta-ifx-android — Plan

Derived from `cajeta-ifx-android-spec.md`. Outline numbering `1` / `a` / `1`.
Checkbox legend: `[x]` done · `[~]` partial · `[ ]` not started.

## 1. Window + Surface backend
Android window creation via NDK ANativeWindow / GameActivity, producing the opaque `Surface` for `VK_KHR_android_surface`.

   **TDD**
   a. [ ] `probe()` returns true under Android, false elsewhere (env-forced).
   b. [ ] `createWindow` + `surfaceOf` yield a non-headless `Surface` with the requested extent.

   **Deliverables**
   a. [ ] `AndroidWindowBackend implements WindowBackend` + self-registration.
   b. [ ] FFI bindings for NDK ANativeWindow / GameActivity (create / destroy / poll) + the surface handle.

   **Acceptance Criteria**
   a. [ ] A window opens on Android and the gfx swapchain presents to its `Surface`.

## 2. Input
   **TDD**
   a. [ ] Synthetic AInputQueue / GameActivity (touch + gamepad) events surface as `WindowEvent` / `InputDevice` state.

   **Deliverables**
   a. [ ] keyboard/mouse via the window queue; gamepads via AInputQueue / GameActivity (touch + gamepad).

   **Acceptance Criteria**
   a. [ ] Input round-trips on a real Android device.

## 3. Audio
   **TDD**
   a. [ ] An output stream plays a known buffer; capture reads a known input.

   **Deliverables**
   a. [ ] `AudioBackend` over AAudio (+ OpenSL ES fallback) (output + capture).

   **Acceptance Criteria**
   a. [ ] Audio output is audible / captured PCM matches a reference within tolerance.

## 4. Registration & dispatch
   **TDD**
   a. [ ] Registers at load; the dispatcher selects it on Android; `CAJETA_IFX_*` override works.

   **Deliverables**
   a. [ ] static-init registration into `BackendRegistry`; `priority()` == 100.

   **Acceptance Criteria**
   a. [ ] On Android with this library linked, `ifx` binds it with no app changes.

## Dependencies & sequencing
- Gated on the stdlib `cajeta.ifx` contract landing (`runtime/src/cajeta/ifx/`).
- The Surface/WSI hand-off is gated on the gfx swapchain (`cajeta.gpu.gfx`, spec Part GP-1 §4.2).
- Versioned independently of the other backends (own Android API cadence).

---

## Appendix A — Binding & capability-gap work (from vendor research)

### 5. Binding (C NDK + JNI airlock)
   **TDD**
   a. [ ] `ANativeWindow` arrives via the GameActivity glue (`APP_CMD_INIT_WINDOW`); Vulkan surface
      built with `vkCreateAndroidSurfaceKHR`.
   b. [ ] JNI round-trip: request `RECORD_AUDIO`; show/hide the IME via GameTextInput.

   **Deliverables**
   a. [ ] GameActivity integration (+ its `android_native_app_glue` fork → `android_main()`); input
      drained via `android_app_swap_input_buffers()`; Paddleboat gamepad; Swappy frame pacing.
   b. [ ] Oboe/AAudio output + capture (low-latency MMAP/EXCLUSIVE requested, graceful fallback).
   c. [ ] A **Java/Kotlin companion + JNI glue** for soft-keyboard text, `RECORD_AUDIO` permission,
      and lifecycle dialogs.

   **Acceptance Criteria**
   a. [ ] Renders to the SurfaceView; gamepad + low-latency audio work; permission flow via JNI.

### 6. Capability gaps & fallbacks (vs spec §9.7)
   **TDD**
   a. [ ] `APP_CMD_TERM_WINDOW` mid-run → swapchain released; `INIT_WINDOW` again → recreated
      (the canonical surface-lost/recreated case).

   **Deliverables**
   a. [ ] lifecycle-driven loop off the Activity lifecycle; single fullscreen surface; surface
      vanish/return mapped to `ifx` surface-lost/recreated events.
   b. [ ] touch floor + optional gamepad via `supports()`; IME text (not raw keys); runtime audio
      latency query with Oboe fallback chain; mic permission via the contract's permission state.

   **Acceptance Criteria**
   a. [ ] App survives surface destroy/recreate and background/foreground without leaking the
      swapchain; degrades cleanly when MMAP low-latency is unavailable.
