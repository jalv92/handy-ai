# Linux build notes (handy-ai fork)

Deltas from upstream `BUILD.md` discovered while building this fork on Ubuntu 24.04 / WSL2.
Follow these in addition to the README/BUILD.md steps.

## 1. System packages

The upstream list is missing a few that the build needs. Full list (Ubuntu/Debian):

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential pkg-config cmake patchelf autoconf automake libtool \
  libasound2-dev libssl-dev libvulkan-dev vulkan-tools glslc \
  libgtk-3-dev libwebkit2gtk-4.1-dev libayatana-appindicator3-dev librsvg2-dev \
  libgtk-layer-shell0 libgtk-layer-shell-dev \
  libevdev-dev libudev-dev libxdo-dev libxkbcommon-dev
```

(`libevdev-dev` is required by `evdev-sys`; `libxdo-dev` by `enigo`. Both are missing from upstream BUILD.md.)

## 2. libclang for bindgen (whisper-rs-sys)

`whisper-rs-sys` runs `bindgen`, which needs `libclang`. Instead of `apt install libclang-dev`
you can use a userspace copy and point bindgen at it:

```bash
pip install --user --break-system-packages libclang   # installs ~/.local/.../clang/native/libclang.so
export LIBCLANG_PATH="$HOME/.local/lib/python3.12/site-packages/clang/native"
```

(Or `sudo apt-get install libclang-dev` and skip `LIBCLANG_PATH`.)

## 3. Vulkan disabled on Linux

`src-tauri/Cargo.toml`: the Linux `transcribe-rs` dependency now uses `whisper-cpp` (CPU) instead of
`whisper-vulkan`. Reason: `whisper-rs 0.16`'s Vulkan backend imports `ggml_backend_vk_*` symbols the
bundled `whisper.cpp` no longer exports, so the Vulkan build breaks. CPU whisper + the ONNX/Parakeet
engine still work (Parakeet is the recommended default and is CPU-optimized). WSL has no GPU
passthrough anyway. To re-enable GPU later, bump `whisper-rs`/`transcribe-rs` to compatible versions.

## 4. Updater artifacts disabled

`src-tauri/tauri.conf.json`: `bundle.createUpdaterArtifacts = false`. The upstream auto-updater points
at the original author's release server and requires a signing key. A local fork build doesn't need it;
disabling it lets `tauri build` finish cleanly without `TAURI_SIGNING_PRIVATE_KEY`.

## 5. Build command

```bash
bun install
LIBCLANG_PATH="$HOME/.local/lib/python3.12/site-packages/clang/native" \
  bun run tauri build --bundles deb
```

Artifacts:
- binary: `src-tauri/target/release/handy`
- installer: `src-tauri/target/release/bundle/deb/Handy_<version>_amd64.deb`

Install: `sudo apt install ./src-tauri/target/release/bundle/deb/Handy_*_amd64.deb`
