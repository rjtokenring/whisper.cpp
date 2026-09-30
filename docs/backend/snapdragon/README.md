# Snapdragon-based devices (Hexagon NPU)

whisper.cpp uses the ggml Hexagon backend (`ggml/src/ggml-hexagon`), the same one used by llama.cpp.
The HTP device is registered as a GPU-type device, so whisper tools use it by default (`-ng` falls back to CPU,
`-dev N` selects the device index).

## Setup

The cross-compilation toolchain images are provided by the
[Qualcomm Snapdragon Toolchain registry](https://github.com/snapdragon-toolchain):

* **Android toolchain**: `ghcr.io/snapdragon-toolchain/arm64-android:v0.7`
* **Linux toolchain**: `ghcr.io/snapdragon-toolchain/arm64-linux:v0.7`

Only Docker is required on the host.

## How to Build

### Using build.py script (Recommended)

`scripts/snapdragon/build.py` copies `docs/backend/snapdragon/CMakeUserPresets.json` to the repo root,
runs the build in the toolchain container, installs into `pkg-TARGET/whisper.cpp` and optionally pushes it to the device.

Linux target (e.g. Arduino VENTUNO Q), deployed to `~/whisper.cpp` on the device:
```
$ ./scripts/snapdragon/build.py --target linux:user@host --push
```

Android target:
```
$ ./scripts/snapdragon/build.py --target adb --push
```

Use `--toolchain-version` to select a different container tag and `--no-docker` to build natively.

### Manual CMake Build

```bash
~/src/whisper.cpp$ docker run -it --rm -u $(id -u):$(id -g) --volume $(pwd):/workspace --platform linux/amd64 ghcr.io/snapdragon-toolchain/arm64-linux:v0.7

[d]/workspace> cp docs/backend/snapdragon/CMakeUserPresets.json .
[d]/workspace> cmake --preset arm64-linux-snapdragon-release -B build-snapdragon
[d]/workspace> cmake --build build-snapdragon -j $(nproc)
[d]/workspace> cmake --install build-snapdragon --prefix pkg-snapdragon/whisper.cpp
```

The package contains `bin/` (whisper tools) and `lib/` (ggml, whisper and the `libggml-htp-v*.so` DSP skels).

## How to Run

`scripts/snapdragon/run.py` sets `LD_LIBRARY_PATH` / `ADSP_LIBRARY_PATH` and maps the `--hex-*` options to the
`GGML_HEXAGON_*` environment variables:

```
$ ./scripts/snapdragon/run.py --target linux:user@host -- whisper-cli -m ~/models/whisper/ggml-base.bin -f ~/models/whisper/jfk.wav
$ ./scripts/snapdragon/run.py --target linux:user@host -- whisper-bench -m ~/models/whisper/ggml-base.bin
$ ./scripts/snapdragon/run.py --target linux:user@host --hex-opfilter MUL_MAT -- whisper-cli ...
```

Look for `using HTP0 backend` in the log.

## Status

Tested on Arduino VENTUNO Q (QCS8275, HTP v75), `whisper-bench`, encoder time, 4 threads:

| Model | CPU F16 | CPU Q8_0 | HTP F16 |
|-------|---------|----------|---------|
| base  | 1293 ms | 842 ms   | 197 ms  |
| small | 4467 ms | 2610 ms  | 452 ms  |

Known limitations:

* Use **F16** models. `Q8_0` MUL_MAT on HTP with the whisper encoder shapes (n=1500) crashes the DSP with HMX
  (`INTERNAL-ERROR`) and gives wrong results with HVX (`--hex-mm-select 1`).
  `Q5_0`/`Q5_1` are not supported by HTP: MUL_MAT runs on CPU, slower than a pure CPU run (`-ng`).
* Flash attention must stay enabled (default). With `-nfa` the V cache copy (transposed f32 -> f16) is not supported by HTP.
