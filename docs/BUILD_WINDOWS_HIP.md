# Building the HIP engine from source on Windows

[AMD_HIP.md](AMD_HIP.md#windows) covers the normal Windows path: download the release, double-click
`START-HERE.bat`, and setup fetches the ready-made `strata-windows-x64-hip.zip`. This page covers building that
same zip yourself, which is what you want when the card is not in the released archs, when you are changing engine
code, or when you want the binary built for one arch instead of eight.

The build needs **no AMD GPU, no ROCm install and no admin rights**. ROCm comes from AMD's TheRock Python wheels
into a private venv in the repo.

Measured on: Windows 11 26100, AMD Radeon RX 7700 XT (gfx1101, 12 GB), Ryzen Threadripper PRO 7995WX (96 cores),
63 GB RAM, engine 0.1.43, ROCm 10.2.0a20260930.

## What you need

| | this machine had |
| --- | --- |
| Visual Studio Build Tools, C++ workload | VS 18 Community, MSVC 14.51.36231, Windows SDK 10.0.26100 |
| Python 3.10+ reachable as `py -3` or `python` | `py -3` → 3.13.15 |
| git | 2.55.0 |
| disk | ~9 GB (see [Disk](#disk)) |

CMake and Ninja come with the Visual Studio C++ workload; the ones on PATH here were CMake 4.3.1 and Ninja 1.13.2.

The AMD driver is only needed to *run* the engine, not to build it.

## Build

One command from the repo root:

```bat
tools\hip\build_windows.bat
```

That builds every arch the release ships (`gfx1100;gfx1101;gfx1102;gfx1200;gfx1201;gfx1030;gfx1031;gfx1151`). To
build only your own card's arch, set `STRATA_HIP_ARCHS` first — this is most of the compile time:

```bat
set STRATA_HIP_ARCHS=gfx1101
tools\hip\build_windows.bat
```

Find your arch in [AMD_HIP.md](AMD_HIP.md) (RX 7700 XT / 7800 XT = `gfx1101`, RX 7900 XT / XTX = `gfx1100`,
RX 9070 / R9700 = `gfx1201`, RX 9060 XT = `gfx1200`, RX 6800 / 6900 = `gfx1030`), or run
`strata-device.exe --selftest` from any existing install.

Add `tests` as the first argument to build the ctest targets as well; those need an AMD GPU to run.

The script does four things:

1. installs ROCm into `.rocm-win` from `https://nightly.repo.amd.com/rocm/whl-next/` — the compiler, the HIP
   runtime, hipBLAS / rocBLAS / hipBLASLt, and one device package per arch. Skipped when `.rocm-win\strata-rocm.txt`
   already records the same version and arch list.
2. finds Visual Studio with `vswhere` and runs `vcvars64.bat`. ROCm's clang compiles the host code too, targeting
   the MSVC ABI with MSVC's headers and linker — CMake refuses to mix `cl.exe` with Clang for HIP.
3. configures and builds with Ninja into `build-hip-win`.
4. packages `dist\strata-windows-x64-hip.zip` with the two programs, the ROCm DLLs they load, and the licenses.

Other settings, all optional: `STRATA_ROCM_VERSION`, `STRATA_ROCM_INDEX`, `ROCM_VENV`, `BUILD_DIR`, `DIST_DIR`,
`STRATA_GGML_DIR` (a llama.cpp checkout at the pinned commit; CMake fetches it otherwise).

## What you get

```
build-hip-win\strata.exe              21.9 MB
build-hip-win\strata-device.exe       75 KB
dist\strata-windows-x64-hip.zip       127 MiB  (379 MiB unpacked)
dist\strata-windows-x64-hip\          the same, unpacked
```

The zip is the same artifact setup downloads, so anything that consumes a release engine consumes this.

## Check it

From the unpacked `dist\strata-windows-x64-hip`:

```bat
strata-device.exe --selftest
```

On the RX 7700 XT that printed:

```
device 0: AMD Radeon RX 7700 XT
  HIP arch            gfx1101 wave32 (compiled for gfx1101)
  multiprocessors     24
  VRAM total / free   11.984 GiB / 10.906 GiB
memory plan  --max-context 20480
  expert cache        5.675613 GB  (4105 slots of 1,382,400 B)
  plan + KV vs free   FITS
selftest: over-allocation refused as required
strata-device selftest OK
```

`compiled for gfx1101` is the line that confirms the binary matches the card. A mismatch there is the first thing
to check if the engine later fails on a prompt.

## Use the build instead of the released engine

`setup.py` takes a local folder for the ready-made engine, so point it at `dist`:

```bat
START-HERE.bat --yes --backend hip --prebuilt D:\llama.cpp\strata\dist --family qwen --model IQ2_XS
```

Add `--gguf-dir DIR` to use GGUF shards you already have instead of downloading ~40-60 GB, and `--no-start` to
set up without starting the server. `--check` checks the PC and exits.

## Running strata.exe directly

`strata.exe` needs the ROCm DLLs in `rocm\bin`, which are **not** next to it. Without them it exits immediately
with `0xC0000135` (`STATUS_DLL_NOT_FOUND`) and prints nothing:

```bat
set PATH=%CD%\rocm\bin;%PATH%
strata.exe --help
```

`strata-device.exe` does run from the top level, because the DLLs it needs are duplicated there. So a passing
`strata-device --selftest` does not prove `strata.exe` will start.

Note that `strata.exe generate` takes a **pack directory** (`--pack`) and pretokenized input (`--tokens`), not a
GGUF and not text. Setup builds the pack from the GGUF; the server in `serve/` is the thing that takes text.

## Disk

| | |
| --- | --- |
| `.rocm-win` (ROCm venv) | 7.96 GB |
| `build-hip-win` | 0.24 GB |
| `dist` | 0.49 GB |

`.rocm-win` is reusable: a later build with the same ROCm version and arch list skips the download. Building all
eight archs instead of one adds a device package per arch.

## Build time

3.1 minutes end to end on the 96-core Threadripper for `gfx1101` alone, from an empty `.rocm-win`, of which the
ROCm wheels were 1.4 GB downloaded in about 12 s at 70-100 MB/s (unpacking and `rocm-sdk init` take longer than
the transfer). The compile itself is 142 Ninja targets. Expect considerably longer on fewer cores;
[STRIX_HALO.md](STRIX_HALO.md) quotes about 25 minutes at `-j 8` on Linux. Building all eight archs multiplies the
device-code compile, not the host compile.

## Notes

- The configure step prints `Strata HIP: gfx1101 was validated by community reports (docs/AMD_HIP.md)` for the
  community-validated archs. It is a note, not a warning.
- No errors on this build. Warnings are per translation unit (73 summary lines, the largest `242 warnings
  generated when compiling for host`); most are deprecation notices from `ucrt` headers and unused `small_mmvq`
  template instantiations in `native_mmvq.cu`.
- A cosmetic `'vswhere.exe' is not recognized` can appear before the CMake output without stopping the build. The
  MSVC toolchain was still picked up here — the compile log shows `VC\Tools\MSVC\14.51.36231\include` and
  `Windows Kits\10\include\10.0.26100.0`. If the build really cannot find Visual Studio it stops with
  `Visual Studio Build Tools with the C++ workload are needed` before CMake runs at all.
- `-- strata: pack/full/experts.bin absent - the expert arena load test is NOT registered` during configure is
  expected without a prepared model.
