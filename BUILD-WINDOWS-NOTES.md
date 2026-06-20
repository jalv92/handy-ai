# Windows build (handy-ai fork)

Two ways to get a Windows installer (`.msi` + NSIS `.exe`).

## Option A — GitHub Actions (recommended, no local toolchain)

A workflow at `.github/workflows/build-windows.yml` builds the Windows installers on
GitHub's `windows-latest` runners (MSVC, WebView2, cmake all preinstalled).

- **Run it:** GitHub → repo **Actions** tab → **Build Windows** → **Run workflow**.
  (Or from a machine with the `gh` CLI: `gh workflow run build-windows.yml`.)
- It also runs automatically when you push a version tag (`git tag v0.8.3 && git push --tags`).
- When it finishes, download the **`handy-windows-installers`** artifact from the run page.
  It contains:
  - `bundle/msi/Handy_<version>_x64_en-US.msi`  (Windows Installer)
  - `bundle/nsis/Handy_<version>_x64-setup.exe`  (NSIS setup)
  - `handy.exe`  (raw binary; the installers are what you distribute)

## Option B — Build on your own Windows machine

### Prerequisites (install once)

```powershell
# Rust (MSVC toolchain)
winget install Rustlang.Rustup
rustup default stable-msvc

# Microsoft C++ Build Tools (MSVC + Windows SDK) — needed by whisper.cpp / native crates
winget install Microsoft.VisualStudio.2022.BuildTools
#   In the VS Build Tools installer, select: "Desktop development with C++"

# Bun
winget install Oven-sh.Bun

# LLVM (provides libclang.dll for bindgen / whisper-rs-sys)
winget install LLVM.LLVM

# CMake (whisper.cpp build)
winget install Kitware.CMake

# WebView2 runtime is preinstalled on Windows 11; on older Windows install:
#   winget install Microsoft.EdgeWebView2Runtime
```

### Build

```powershell
# From the repo root, in PowerShell:
$env:LIBCLANG_PATH = "C:\Program Files\LLVM\bin"   # so bindgen finds libclang
bun install
bun run tauri build
```

Output:
- `src-tauri\target\release\bundle\msi\Handy_<version>_x64_en-US.msi`
- `src-tauri\target\release\bundle\nsis\Handy_<version>_x64-setup.exe`

## Fork-specific notes (same reasoning as Linux)

- **Whisper Vulkan disabled** (`src-tauri/Cargo.toml`, Windows target): uses `whisper-cpp`
  (CPU) + `ort-directml` (GPU-accelerated Parakeet on Windows). `whisper-rs 0.16`'s Vulkan
  backend imports `ggml_backend_vk_*` symbols the bundled `whisper.cpp` no longer exports,
  which breaks the build. Parakeet (the recommended default engine) still gets DirectML GPU
  acceleration via `ort-directml`.
- **Auto-updater artifacts disabled** (`tauri.conf.json` `createUpdaterArtifacts:false`) so the
  build finishes without a `TAURI_SIGNING_PRIVATE_KEY`.
- **libclang** is required for `whisper-rs-sys` (bindgen) — hence the LLVM install + `LIBCLANG_PATH`.
