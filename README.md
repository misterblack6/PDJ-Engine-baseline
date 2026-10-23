# PDJ-Engine-baseline

A build of [Project DJ Engine (PDJE)](https://github.com/misterblack6/PDJ-Engine-baseline) configured for **older CPUs**: the compiler baseline is **SSE4.2 with no AVX/AVX2**. Based on engine version **0.9.4**.

PDJE is a modular C++20 DJ/rhythm-game engine with a database-backed playback and editing core, device/MIDI input, note judgment, and a C ABI for language bindings (Python, C#, and the Godot plugin).

---

## Why this baseline

Upstream builds enable `/arch:AVX2` (MSVC) and `-march=haswell -mavx2 -mfma` (GCC/Clang), which crashes with `0xC000001D` (illegal instruction) on pre-Haswell/Pre-SSE4.2-era machines. This fork targets a portable instruction set instead:

| | Upstream | This baseline |
| --- | --- | --- |
| MSVC (x64) | `/arch:AVX2` | *(flag removed — MSVC x64 defaults to SSE2, a subset of SSE4.2)* |
| GCC/Clang | `-march=haswell -msse4.1 -mavx -mavx2 -mfma` | `-msse4.2 -mpopcnt` |
| Highway targets | `scalar,sse4,avx,avx2` | `scalar,sse4` |

All of these are set in [`cmakes/SETcmake.cmake`](cmakes/SETcmake.cmake).

**Verification:** object files and binaries built from this repo's own sources contain **zero** AVX/AVX2/AVX-512 instructions (checked with `dumpbin /DISASM`, validated against a control binary built with `/arch:AVX2`). Some third-party static libraries (e.g. OpenSSL, ONNX Runtime) still ship AVX code, but only behind **runtime CPUID dispatch**, so those paths are never taken on an SSE4.2-only CPU.

> Requirement: any x86-64 CPU with SSE4.2 (Intel Core 2 / Nehalem and newer, AMD Bulldozer and newer).

---

## Requirements

- **Windows:** Visual Studio 2022+ (MSVC, toolset v145), CMake ≥ 3.23, Ninja, Python 3, [Conan 2](https://conan.io/)
- **Linux:** Clang, CMake ≥ 3.23, Ninja, Conan 2, plus `libevdev-dev libsystemd-dev libnuma-dev libasound2-dev libwayland-dev pkg-config`
- **macOS:** AppleClang, CMake ≥ 3.23, Ninja, Conan 2, plus `brew install swig`

```bash
pip install conan
conan profile detect
```

---

## Building

### Windows

```bat
BuildInitwithConan.bat . dynamic RelWithDebInfo
windows_conf_and_build.bat RelWithDebInfo %NUMBER_OF_PROCESSORS% on
```

Use `Release` instead of `RelWithDebInfo` for a build without tests. Outputs land in `build/` (8 DLLs — `PDJE`, `CPDJE`, the INPUT/JUDGE modules and their CPDJE variants, `PDJE_UTIL`, `PDJE_LOG_RUNTIME` — plus 21 EXEs when tests are enabled).

### Linux / macOS

```bash
bash BuildInitwithConan.sh . Release
cmake --preset linux-release        # or: macos-release / linux-relwithdebinfo / ...
cmake --build --preset linux-release
```

Presets live in [`CMakePresets.json`](CMakePresets.json): `windows-release`, `windows-relwithdebinfo`, `linux-release`, `linux-relwithdebinfo`, `macos-release`, `macos-relwithdebinfo`.

---

## Testing

Tests are enabled by the `*-relwithdebinfo` presets (`PDJE_TEST=ON`).

```bash
ctest --preset windows-relwithdebinfo   # label: unit
ctest --preset linux-relwithdebinfo
```

Stable CTest labels for targeted runs: `core`, `input`, `judge`, `util`, `cabi`, `onnxruntime`, `unit` — see [`AGENT_DOCS/TEST_MAP.md`](AGENT_DOCS/TEST_MAP.md) for the full mapping. Manual/dev programs (`testEditor`, `testInput`, `testJudge`, `testOnnxLoad`, …) are built with `PDJE_DEV_TEST=ON` and are not registered with CTest.

Latest result on Windows x64 (RelWithDebInfo): **168/169 passed**. The single failure (`unit.util::stft request opencl backend matches serial output when available`) is an OpenCL/iGPU scale mismatch unrelated to the CPU baseline.

---

## Architecture

```
authored music + track data
            |
            v
         PDJE core -----> playback / editor
            |
            | borrowed core data line
            v
          judge <------- PDJE_Input
            ^             borrowed input data line
            |
     rules, rails, notes, callbacks
```

| Surface | C++ header | C ABI header |
| --- | --- | --- |
| Core (database, playback, editor) | `include/core/interface/PDJE_interface.hpp` | `include/core/interface/CPDJE_interface.h` |
| Input (device/MIDI) | `include/input/PDJE_Input.hpp` | `include/input/CPDJE_Input.h` |
| Judge (rules, rails, notes) | `include/judge/PDJE_Judge.hpp` | `include/judge/CPDJE_Judge.h` |
| Utilities | `include/util/PDJE_Util.hpp` | — |

Sources are listed explicitly in `cmakes/src/**` — adding a file to disk does not add it to a target. Public headers are installed via `cmake --install`.

### Bindings

- **SWIG Python / C#:** `PDJE_swig.i`, `swig_python/`, `swig_csharp/` (enable with `PDJE_SWIG_BUILD=ON`)
- **Godot:** the [PDJE-Godot-Plugin](https://github.com/Rliop913/PDJE-Godot-Plugin) wrapper repo consumes these DLLs. The engine links Highway **statically** (`hwy.lib` from Conan), so no `hwy.dll` needs to be shipped alongside them.

---

## Dependencies (Conan)

`spdlog`, `annoy`, `capnproto`, `highway`, `libgit2`, `zlib`, `rocksdb`, `botan`, `rapidfuzz` — plus ONNX Runtime, SQLite, miniaudio, SoundTouch, and LibreMIDI. See [`conanfile.txt`](conanfile.txt).

---

## License

[Apache License 2.0](LICENSE)
