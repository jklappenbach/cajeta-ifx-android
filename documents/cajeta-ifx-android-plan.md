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
