---
date: 2026-10-06
title: Benchmarking twelve VLAs on an Orin NX for vla.cpp
slug: benchmarking-twelve-vlas-on-an-orin-nx
---

# Benchmarking twelve VLAs on an Orin NX for vla.cpp

vla.cpp (the llama.cpp-style engine for vision-language-action models) had benchmark reports for the Orin Nano Super and the AGX Orin, and nothing in between. The Orin NX 16 GB is the board most hobby arms end up on. This report fills the gap and is merged upstream as vla.cpp `#36`.

<!-- more -->

## Protocol

The project's own template: 3 warm-ups, 20 repetitions, best of 3 processes, a flag screen, then a full re-run of the fastest flag set. JetPack 6.2, CUDA 12.6, `MAXN_SUPER`, built on the board with `-DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=87`. Twelve of thirteen models; OpenVLA-OFT does not fit in memory. Every GGUF's sha256 checked against the Hub. One evening, 18:10 to 20:57, no errors.

## Headline, minimum ms per inference at the defaults

| | Octo | TurboVLA | VLA-JEPA | GR00T N1.7 | BitVLA | SmolVLA | π0 | Evo-1 |
|---|--:|--:|--:|--:|--:|--:|--:|--:|
| Orin Nano Super | 31.9 | 89.3 | 193.9 | 271.7 | 335.0 | 462.8 | 845.2 | 1011.5 |
| **Orin NX 16 GB** | **27.4** | **79.7** | **176.7** | **251.4** | **269.1** | **411.2** | **739.6** | **893.2** |
| AGX Orin | 17.5 | 36.0 | 94.6 | 132.7 | 133.0 | 218.3 | 350.3 | 434.6 |

Every model lands between the two existing boards, as it should. The fastest flags match the Nano report except GR00T N1.6, where `--weight-dtype f16` passes the 3 % rule on this board.

## Two things worth knowing

- On this L4T release, `getrusage` peak RSS does not include CUDA buffers (π0 reports 483 MiB here against 5 961 MiB on the Nano). I kept the template column as measured and added a whole-board `tegrastats` column with the idle baseline; the maintainer chose to drop the memory columns.
- Throttling check: GPU clock stayed at 1 162–1 172 MHz, junction temperature ≤ 83 °C under the full sweep, so the numbers are not thermally capped.

## Why it matters here

These are engine latencies on fixed inputs. The same board running SmolVLA on a real SO-101 through the full client measured 238–390 ms per chunk, which is the number the control loop actually sees. Both numbers are on the issue tracker (`#31`) so the next person with this board does not have to guess.
