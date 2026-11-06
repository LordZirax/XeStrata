# How this record's engine was built

**It was not built.** This measurement used the official prebuilt Windows package,
`XeStrata-0.2.2-1-windows-x64.7z`, unpacked as-is. No compiler, no toolchain, no local patches.

That is worth stating because it is the point of this entry: on Windows the engine needs no build at
all, and the numbers in `matrix.jsonl` come from that untouched package.

## What the Windows package needs

- **.NET 10 runtime** — `Microsoft.NETCore.App` and, for the GUI `xestrata.exe`,
  `Microsoft.WindowsDesktop.App`. 10.0.14 was installed here.
- **Visual C++ Redistributable (x64)** — `vcruntime140.dll`, `msvcp140.dll`. The GUI checks and stops
  with a clear message if it is missing.
- **The GPU driver** — Intel `32.0.101.8976` here.
- Level Zero and the SYCL runtime ship **inside** the package; nothing system-wide was needed.

Confusing detail, worth passing on: `dotnet --version` fails with *"The application '--version' does
not exist"* when only the runtime is installed (no SDK). That is not a missing runtime. Use
`dotnet --list-runtimes`.

## What was deliberately NOT installed

- **Intel oneMKL.** The setup offered it and it was declined, so XeStrata's own kernels did the matrix
  products. These are *free-build* numbers, unlike the contrib-llvm record on 4x B70.
- **No model download.** The pack was prepared from GGUF files already on disk.

## What the first start cost

The setup compiled the GPU code for this model once: **784 s**. After that, every start reads the
weights and fills the expert cache in about 16 s (23.42 GiB at 2.62 GiB/s). The first two requests
after a fresh start are slow (prefill 4-5 tok/s for ~30 s) while that settles — see the note in the
main README, and do not report those as steady state.
