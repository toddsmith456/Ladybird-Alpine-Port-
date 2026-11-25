# Ladybird Browser — Alpine Linux Port (`2026.47.7-r1`)

Prebuilt Alpine Linux (`edge`, `x86_64`) `.apk` package, reproducible `APKBUILD`, and musl compatibility patches for **Ladybird Browser `2026.47.7`** (built from the official source tarball at <https://ladybird.org/download/>).

## Contents

- `ladybird-2026.47.7-r1.apk` — Signed Alpine Linux `x86_64` package (~67 MB compressed, ~178 MB installed) with full musl sandbox, ICU, and stack-size fixes
- `builder@local-6b061501.rsa.pub` — RSA public key used to sign `ladybird-2026.47.7-r1.apk`
- `ladybird-apkbuild/APKBUILD` — Alpine `abuild` recipe (`pkgrel=1`) with verified `sha512sums`
- `ladybird-apkbuild/0001..0007-*.patch` — Musl / Alpine build, sandbox, and runtime patches
- `SHA256SUMS` — SHA-256 checksums for all files

## Installation

### Alpine Linux `edge` (`x86_64`)

Install the signing key:

```sh
sudo cp builder@local-6b061501.rsa.pub /etc/apk/keys/
```

Install or upgrade the package:

```sh
sudo apk add ./ladybird-2026.47.7-r1.apk
```

### Alpine Linux `3.24` (`x86_64`)

If you already installed `-r0` (so `simdutf>=9.4` and `Qt 6.12` from `edge` are already installed), upgrade in place:

```sh
sudo cp builder@local-6b061501.rsa.pub /etc/apk/keys/
```

```sh
sudo apk add ./ladybird-2026.47.7-r1.apk
```

For a fresh install on Alpine `3.24`:

```sh
sudo cp builder@local-6b061501.rsa.pub /etc/apk/keys/
```

```sh
sudo apk add \
  -X https://dl-cdn.alpinelinux.org/alpine/edge/main \
  -X https://dl-cdn.alpinelinux.org/alpine/edge/community \
  'simdutf>=9.4' \
  'qt6-qtbase>=6.12' 'qt6-qtbase-x11>=6.12' 'qt6-qtbase-wayland>=6.12' 'qt6-qtpositioning>=6.12' \
  icu-data-full \
  ./ladybird-2026.47.7-r1.apk
```

Then run (with the sandbox enabled — no `--disable-sandbox` needed):

```sh
ladybird
```

## What's Fixed in `-r1`

1. **Full Linux Sandbox (`Seccomp` + `Landlock` + `FileBroker`) on musl (`0006-libsandbox-musl-seccomp-syscalls.patch`):**
   - Allows `ioctl(..., TIOCGWINSZ)` (used by musl's `isatty()`), eliminating `SIGSYS (31)` crashes in `WebContent` and `RequestServer`.
   - Allows `pwritev2`, `preadv2`, `pwritev`, `preadv`, and `fadvise64` (musl `1.2.6` implements `pwrite()` via `pwritev2(..., RWF_NOAPPEND)`).
   - Allows/brokers `lstat` alongside `stat`/`newfstatat` (musl calls `__NR_lstat` directly on x86_64).
   - Allows `tkill` (musl `raise()`/`pthread_kill()`) under Landlock signal scope (or for the current thread) and refuses `fchown` with `EPERM`.
   - Grants read-only access to `/usr/share/icu`, `/usr/share/zoneinfo`, and `/etc/localtime` inside `Sandbox::restrict_filesystem()` so Alpine's unbundled `libicudata.so` stub can load `/usr/share/icu/78.3/icudt78l.dat`.
2. **Musl Main-Thread Stack Bounds for C++ and Rust (`0007-ak-musl-stack-bounds.patch`):**
   - Interposes `pthread_getattr_np` in `liblagom-ak.so.0` (and patches `AK/Rust/src/stack.rs` + `AK/StackInfo.cpp`) so musl reports the full `RLIMIT_STACK` (8 MiB minus guard page) instead of only the ~128 KiB initially committed stack pages. Fixes `LibJS/Rust` parser/RegExp panics (`pattern is nested too deeply` at `parser/mod.rs:371`) on complex JS sites like GitHub.
3. **8 MiB Default Thread Stack (`PT_GNU_STACK` = `0x800000`):**
   - Sets `PT_GNU_STACK.p_memsz = 0x800000` (`-Wl,-z,stack-size=8388608`) on all 12 Ladybird executables so musl worker threads get 8 MiB stacks instead of musl's 128 KiB default.
4. **Full ICU Data Dependency (`icu-data-full`):**
   - Adds `icu-data-full` to `depends` so `LibUnicode/Segmenter` never hits `U_MISSING_RESOURCE_ERROR` with Alpine's minimal `icu-data-en`.

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
6. **`0006-libsandbox-musl-seccomp-syscalls.patch`** — Musl seccomp syscalls (`TIOCGWINSZ`, `pwritev2`/`preadv2`/`pwritev`/`preadv`, `fadvise64`, `lstat`, `tkill`, `fchown`) and Landlock/FileBroker read-only access for `/usr/share/icu`, `/usr/share/zoneinfo`, and `/etc/localtime`.
7. **`0007-ak-musl-stack-bounds.patch`** — Fixes `AK/StackInfo.cpp` (`8 * MiB` for `RLIM_INFINITY` + musl `pthread_getattr_np` main-thread stack expansion) and `AK/Rust/src/stack.rs` musl `RLIMIT_STACK` handling.

## Verification (`-r1`)

- Verified `apk add` upgrade from `-r0` -> `-r1` with RSA signature check on Alpine `edge` (`x86_64`, `musl 1.2.6`).
- Verified `PT_GNU_STACK` = `0x800000` (8 MiB) on all 12 executables and 100% ABI symbol compatibility on `liblagom-ak.so.0.1.0` and `liblagom-sandbox.so.0.1.0`.
- Verified live sandboxed browsing (**sandbox enabled, no `--disable-sandbox`**) across GitHub (`800x4241` PNG), DuckDuckGo (`1000x3072` PNG), Wikipedia (`800x3297` PNG), Reddit (`800x644` PNG), and Hacker News (`804x1262` PNG) — plus persistent profiles and FileBroker fallback — with zero `SIGSYS` violations, zero Rust panics, and zero ICU errors.
