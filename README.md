# cajeta-ifx-android

`dev.cajeta.ifx.android` — the **Android** backend for **ifx**, the Cajeta interface-framework
facade for window / input / audio. It implements the `ifx` backend SPI by FFI-binding Android system
APIs and registers itself so the `ifx` dispatcher selects it at runtime on Android.

Part of the Cajeta graphics stack: stdlib **`ifx`** owns the portable, write-once contract; this
repo is one **optional, vendor-versioned backend**, pulled in (target-conditionally) via the
**`cajeta-ifx-backend`** melt. App code never imports this directly — it codes against `ifx`.

## What it binds
- **Window / surface:** NDK ANativeWindow / GameActivity → `VK_KHR_android_surface`
- **Audio:** AAudio (+ OpenSL ES fallback)
- **Input:** AInputQueue / GameActivity (touch + gamepad)

> **Notes.** NDK/JNI; a single ANativeWindow surface; the Activity lifecycle drives the loop.

## Status
**v0.1 — library skeleton** (cajeta build-tool project). No implementation yet — see
[`documents/`](documents/) for the spec and plan.

## Build
```sh
cajeta build      # emits the .cja library
cajeta test       # unit tests
```

## License
MIT © 2026 Julian Klappenbach. See [LICENSE](LICENSE).
