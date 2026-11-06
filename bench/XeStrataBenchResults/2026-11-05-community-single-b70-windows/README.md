# Community benchmark on 1x Intel Arc Pro B70, **native Windows** (SYCL)

Measured on 2026-11-05. **Official upstream release**, `XeStrata-0.2.2-1-windows-x64.7z`, unpacked
as-is: **no local build, no engine patches**. The only difference from the release is the run
configuration documented below. This is, to our knowledge, the **first Windows entry** in this table —
every other record here is Linux.

Model: **GSQ-RCO IQ1_M** (58.41 GB, two shards), `--max-context 32768`, one card, MTP draft layer.
Headline, medians of three runs (prompt 3,223 / 29,365 tokens):
**PP 1,169.4 / 1,978.2 tok/s** and **TG 67.1 / 70.4 tok/s**, `max_tokens` 128, temperature 0.
Needle-in-a-haystack recall: **6/6**.

> **Do not compare these tok/s against the 4x B70 record directly.** That run used **UD-Q4_K_XL** and a
> 256-token output cap; this one uses **IQ1_M** (a far smaller quant) and 128. What is new here is not
> a speed record — it is a **Windows** data point on the same card, with the same harness.

## Hardware and software

All figures below are the engine's own readings, not ours from WMI (which reports 4095 MB of VRAM for
every card here — the 32-bit `AdapterRAM` overflow).

- GPU: **Intel Arc Pro B70** (Xe2), **31 GB** as the engine saw it, **31,831 MiB free at start**,
  PCI `0000:b0:00.0`, used alone. An Arc Pro B50 (engine reading: 16 GB) and the integrated graphics
  were present and idle. The setup reported *"PCIe link not readable"* for the B70.
- CPU: **Intel Core Ultra 9 285HX**, 24 threads. The engine reported **AVX2, no AVX-512**, so the
  AVX-2 expert kernels ran. RAM: **127.4 GiB** visible; the engine's memory guard read 127.4 GiB with
  **110.4 GiB available** at start.
- Board: **ARWSA** (Minisforum MS-02 Ultra). Storage: one NVMe volume for model, pack and MTP.
- OS: **Windows 11 Pro build 26300**, native. **No WSL.** GPU driver Intel `32.0.101.8976`.
  .NET 10.0.14 (Core + WindowsDesktop) and the VC++ Redistributable, as the Windows section asks.
- **Intel oneMKL was NOT installed** — the setup offered it and it was declined, so XeStrata's own
  kernels did the matrix products. These are **free-build** numbers; the 4x B70 record used the
  contrib-llvm build with oneMKL.
- Harness: `bench.py` and `needles.py` from
  [`2026-10-12-community-4x-arc-pro-b70`](../2026-10-12-community-4x-arc-pro-b70), **unmodified**,
  run with CPython 3.14.9, standard library only.

## Run configuration

```
--pack <data>/packs/coder-iq1_m
--native <models>/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf
--ple-gguf <models>/Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf
--expert-profile <app>/data/expert-profile-coder.bin
--expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5
--mtp <data>/mtp/rt --max-context 32768 --kv int8 --vram-free-mib 31799
```

The pack was prepared from GGUF files already on disk — no model download. From the engine's log:

```
native pack: largest blob 2.66 MB, token embedding IQ4_XS in mapped host memory
206 MiB of weights loaded from the pack (494 canonical tensors skipped: served natively)
300 native projection matrices, 2,656.38 MiB of weights
mtp: draft layer 839 MiB of VRAM, draft head over 106,299 tokens (212.9 MiB)
expert cache auto: 31.17 GiB free, 700 MiB reserved -> 12,206 slots; cache 12,288 slots, 23.42 GiB
PCIe split: the share 0.10 of the missed experts
2,350 MiB of VRAM free with everything loaded
```

The setup compiled the GPU code for this model once: **784 s**.

## What the numbers show

| prompt (tokens) | PP tok/s median (min-max) | TG tok/s median (min-max) | drafts accepted |
|---|---:|---:|---:|
| 3,223 | **1,169.4** (1,169.2-1,170.1) | **67.1** (62.2-68.4) | 56/82 · 64/81 · 62/85 |
| 29,365 | **1,978.2** (1,978.0-1,978.7) | **70.4** (63.8-75.7) | 59/84 · 69/87 · 80/99 |

- **Prefill is very stable and improves with length** (1,169 → 1,978, run-to-run within 0.1 %), the
  same shape the 4x record shows (1,111 → 1,869 → 2,233): the batch gets more efficient per token.
- **Decode is flat across the two sizes** (67.1 / 70.4) but **varies ±5-7 tok/s run to run**, so the
  medians matter more than any single run.
- **Draft acceptance ranges 68-81 % per run** (median ratios 76 % and 79 %) with `--spec 4`; that is
  what carries the decode figure.

## Needle-in-a-haystack recall: 6/6

A unique 5-digit code at 10 / 50 / 90 % depth, cold prefill each time (`needles.py`, unmodified):

| prompt tokens | depth 0.1 | depth 0.5 | depth 0.9 |
|---|---:|---:|---:|
| 3,630 | recalled · PP 1,105.1 | recalled · PP 1,216.0 | recalled · PP 1,215.8 |
| 21,597 | recalled · PP 1,924.5 | recalled · PP 1,923.9 | recalled · PP 1,923.3 |

## Limitations

- **One card, one model, one quant.** The 131,072-token size is absent because this run used
  `--max-context 32768` — the RAM headroom on this box, and a second B50 is planned. Sizes above 32k
  are therefore not measured here, not measured-and-failed.
- **Quality is only the needle test.** No perplexity, no task suite. Recall at 32k says nothing about
  answer quality in general.
- **Windows only.** Nothing here says how WSL 2 behaves.
- **Cold start is real and misleads:** the first two requests after a fresh start showed **prefill of
  4-5 tok/s for roughly 30 s** (code compile and cache fill) before settling at four figures.
  `bench.py` discards a warm-up request, so its numbers do not show this — anyone measuring by hand
  will see it, and should not report it as steady state.
- **No BIOS verification.** No log here reads Above-4G Decoding or Re-Size BAR; only the engine's
  "PCIe link not readable" line. Nothing in this record depends on those settings being claimed.

## Files

```
build.json           the package and its mode (prebuilt, free kernels, no oneMKL, no patches)
configurations.json  the engine arguments and the card
environment.json     OS, driver, .NET, what was and was not installed
system.json          CPU, RAM, cards, storage, the engine's memory readings
matrix.jsonl         one row per run: prompt/decode tok/s, ms, drafts
results.csv          the same rows, CSV
summary.json         medians and min-max per size, the output cap and temperature
needles.json         6 cells, all recalled
bench.py needles.py  the harness, unmodified
bench-4k.raw.txt     raw console output of the 3,223-token runs
bench-30k.raw.txt    raw console output of the 29,365-token runs
engine-requests.txt  the engine's own per-request lines (the source of matrix.jsonl)
engine-startup.txt   pack preparation, cache sizing, MTP, VRAM readings, ready line
BUILD.md             why there was no build, what the package needs, what the first start cost
```
