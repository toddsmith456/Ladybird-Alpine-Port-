# Ladybird Browser — Alpine Linux Port (`2026.47.7`)

Prebuilt Alpine Linux (`edge`, `x86_64`) `.apk` package, reproducible `APKBUILD`, and musl compatibility patches for **Ladybird Browser `2026.47.7`** (built from the official source tarball at <https://ladybird.org/download/>).

## Contents

- `ladybird-2026.47.7-r0.apk` — Signed Alpine Linux `x86_64` package (~67 MB compressed, ~178 MB installed)
- `builder-6b04b8be.rsa.pub` — RSA public key used to sign the `.apk`
- `ladybird-apkbuild/APKBUILD` — Alpine `abuild` recipe with verified `sha512sums`
- `ladybird-apkbuild/0001..0005-*.patch` — Musl / Alpine `edge` build patches
- `SHA256SUMS` — SHA-256 checksums for all files

## Installation (Alpine Linux `edge`, `x86_64`)

Verify the package signature and install:

```sh
sudo cp builder-6b04b8be.rsa.pub /etc/apk/keys/
sudo apk add ./ladybird-2026.47.7-r0.apk
```

Or install without adding the signing key:

```sh
sudo apk add --allow-untrusted ./ladybird-2026.47.7-r0.apk
```

Then run:

```sh
ladybird
```

> **Requirement:** Targets **Alpine `edge` (`x86_64`)** because Ladybird `2026.47.7` requires exact `ICU 78.3`, `Qt 6.12`, `SDL3`, `simdutf`, `libavif`, and `mimalloc2`.

## Architecture & Packaging Details

- Installed binaries: `/usr/bin/Ladybird` (with `/usr/bin/ladybird` symlink), `/usr/bin/WebDriver`, and all 10 helper processes in `/usr/libexec/ladybird/` (`AudioServer`, `Compositor`, `Hub`, `ImageDecoder`, `MediaServer`, `RequestServer`, `SystemProbe`, `VideoDecoderServer`, `WebContent`, `WebRTCClient`).
- Bundled private libraries (`liblagom-*`, Skia 148, ANGLE `chromium/7258_13`, Ladybird-patched FFmpeg 9.0.2, `libzmij`) are isolated in `/usr/lib/ladybird/` with `RUNPATH=/usr/lib/ladybird` and `sonameprefix="ladybird:"` so they do not conflict with system `mesa-egl`, `mesa-gles`, or `ffmpeg`.
- Built with `clang21` (`21.1.8`), `mold` (`2.39.1`), `rustc`/`cargo` (`1.100.0`), `cmake` (`4.3.4`), and `ninja` (`1.13.2`).

## Patches Included

1. **`0001-cmake-drop-warnings-as-errors-for-musl-and-rust-1.100.patch`** — Removes `-Werror` (musl `<sys/socket.h>` header warnings in `LibSandbox`) and Rust `-Dwarnings` (Rust 1.100 `Atomic::fetch_update` deprecation in `ak`).
2. **`0002-cmake-cargo-1.100-build-script-output-path.patch`** — Supports Cargo 1.100's `build/<crate>/<hash>/run/root-output` layout in `sync_rust_ffi_header.cmake`.
3. **`0003-libsandbox-musl-ioctl-and-msg-controllen.patch`** — Adds `<linux/ioctl.h>` and casts `CMSG_SPACE(...)` to `socklen_t` in `Libraries/LibSandbox/Seccomp.cpp`.
4. **`0004-libmediacodecs-public-ffmpeg-headers.patch`** — Exports `PkgConfig::AVCODEC`/`AVUTIL` as `PUBLIC` on `LibMediaCodecs` so `Services/SystemProbe` resolves Ladybird's FFmpeg headers.
5. **`0005-libjs-rust-disable-glibc-pkey-on-musl.patch`** — Gates glibc-only `pkey_alloc`/`pkey_mprotect`/`pkey_set` calls in `Libraries/LibJS/Rust/src/jit/executable_memory.rs` on `target_env = "gnu"`, falling back to standard `mprotect` W^X JIT on musl.

## Verification

- Verified `apk add` installation with RSA signature check on Alpine `edge` (`x86_64`).
- Verified `ldd` and `RUNPATH=[/usr/lib/ladybird]` across all 12 installed binaries with zero missing libraries or build-path references.
- Verified headless HTML + JavaScript execution (`--headless=text`) and Skia/PNG rendering (`--headless=screenshot`).
