# Building AnyPS5 on Linux ARM64

The relinker and the system libraries build natively on aarch64. The resulting executable is
x86-64, so running it needs an x86-64 translator on top — this document covers the build, and
[the README](../../README.md) covers running.

## Requirements

Debian 13 aarch64 or similar. The packages that matter:

```bash
sudo apt install build-essential cmake ninja-build git python3 \
                 libvulkan-dev vulkan-tools mesa-vulkan-drivers \
                 libsdl2-dev libssl-dev zlib1g-dev
```

For a native Vulkan driver on Qualcomm hardware:

```bash
sudo apt install mesa-vulkan-drivers    # provides the freedreno / Turnip ICD
```

## Configure and build

```bash
cmake -S . -B build-linux -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build-linux --parallel "$(nproc)"
```

The libraries land in `build-linux/core/libs/libs/`. `core/libs/libs/unpatched/` holds the same
libraries before their post-processing step; use the patched copies.

Building every library takes a while on a tablet. To build only what a title needs:

```bash
cmake --build build-linux --parallel "$(nproc)" --target libSceAgcDriver
```

`libSceAgcDriver` is the graphics driver and the largest target. It pulls in the shader
recompiler, so a change under `core/shader/` rebuilds it.

## Translator

AnyPS5 output is x86-64, so a translator is required to run it on aarch64:

```bash
git clone https://github.com/FEX-Emu/FEX.git
```

Two changes in FEX are needed on ARM64, both in
`FEXCore/Source/Interface/Core/JIT/MemoryOps.cpp`:

- **`LoadMemTSO`** — FEX emits `LDAPUR`, `LDAPR` or `LDAR`, all of which require natural
  alignment. Guest x86-64 code loads from unaligned addresses routinely, and the result is
  `SIGBUS` / `BUS_ADRALN`. Emit a plain `LDR` followed by `DMB ISHLD`.
- **`StoreMemTSO`** — likewise, `STLUR` / `STLR` need alignment. Emit `DMB ISH` followed by a
  plain `STR`.

After the change FEX still provides acquire and release semantics; it just no longer faults on
the unaligned accesses the guest makes.

## Runtime environment

```bash
export LD_LIBRARY_PATH=/path/to/game/libs
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/freedreno_icd.json
ulimit -s 65536
FEX ./eboot
```

`APS5_ALLOW_MISSING=1` lets a title start when an optional library is absent. It hides genuine
problems, so enable it only while diagnosing.

## Troubleshooting

**A window appears but stays black, and audio works.** Check whether draws reach a colour
target. AnyPS5 reports this when tracing is on:

```bash
APS5_TRACE_DRAWS=1 FEX ./eboot 2>&1 | grep -o 'colorTarget=[0-9]*' | sort | uniq -c
```

All-zero means the AGC state decoder found no colour target. Verify the mask defaults described
in [CHANGES-ANYPS5.md](../../CHANGES-ANYPS5.md) are present.

**`param.json not found`.** The game content has to be reachable at `app0/sce_sys/param.json`.

**The process faults inside a library.** FEX reports the start of the translated block, not the
faulting instruction, which hides the real address whenever a block holds more than one
instruction. Forcing one instruction per block makes the report exact at the cost of speed:

```bash
FEX_SINGLE_INSTRUCTION=1 FEX ./eboot
```

**Reading a converted executable.** PS5 executables have no section headers, so `readelf -r`
and friends find nothing. Walk `PT_DYNAMIC` instead.
