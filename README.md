# AnyPS5 — Linux ARM64 adaptation

A fork of [boykopovar/AnyPS5](https://github.com/boykopovar/AnyPS5) focused on running PS5
executables on **Linux aarch64**, using [FEX-Emu](https://github.com/FEX-Emu/FEX) to translate
the x86-64 output and a native Vulkan driver (Turnip / freedreno) for rendering.

中文说明见下方 [中文](#中文)。

Upstream converts a PS5 executable into a native x86-64 ELF that links against reimplemented
system libraries. On ARM64 that ELF has to run under an x86-64 translator, so this fork covers
the things that only break on that path.

## What this fork changes

Everything here is upstream `main` plus a small, reviewable set of commits. See
[CHANGES-ANYPS5.md](CHANGES-ANYPS5.md) for the full reasoning and per-fix patches in `changes/`.

| Fix | Effect |
|---|---|
| `CB_TARGET_MASK` / `CB_SHADER_MASK` reset to `0xffffffff` | On GCN and RDNA these registers reset to all channels enabled. AnyPS5 initialised them to zero, so `hasColorTarget` came out false for every draw and a title that relies on the reset values rendered **nothing at all** — a black window with working audio. |
| Pixel shaders with zero inputs | A shader that declares no interpolants can still name interpolation parameter 0. The range check compared `0 >= 0` and failed the whole shader instead of resolving the reference to a constant. |
| Depth maintenance opt-out | `DB_RENDER_CONTROL` depth copy / resummarize / decompress draws are unimplemented and aborted the process. `APS5_ALLOW_DEPTH_MAINTENANCE=1` lets a 2D title continue, where those draws do not affect output. |

Measured on Balatro (PPSA21402): with the mask fix the number of draws that reach a colour
target went from **0 of 387,183** to **79,709 of 81,935**.

## Running

```bash
# 1. Convert the executable with the relinker from this repository.
#    Output layout: eboot, libs/, app0/ (game content).
./relinker --module-dir Media/Modules PPSAxxxxx-eboot.elf eboot

# 2. Put the reimplemented system libraries next to it.
ln -s . '$ORIGIN'
cp -r <build>/core/libs/libs/* libs/

# 3. AnyPS5 reads app0/sce_sys/param.json, so the game content has to be reachable there.
ln -sf ../<content> app0/<content>

# 4. Run it through the translator.
export LD_LIBRARY_PATH=$PWD/libs
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/freedreno_icd.json
FEX ./eboot
```

Notes that cost time to discover:

- Both `libs/libc.prx` (this project) and `app0/sce_module/libc.prx` (from the game) are
  required. Removing either one breaks loading.
- Save data mounts at `/savedata0`. AnyPS5 backs it with a host directory, so a title that
  expects a file already present in the save container needs it placed there.
- `ulimit -s` matters: the default 8 MiB is not always enough once translation adds frames.

## Verified

| Title | Engine | Status |
|---|---|---|
| DOOM (2016) | id Tech 6 | Runs. Menu at 59.94 FPS, ~35 FPS in game |
| Balatro (PPSA21402) | LÖVE / Lua | Renders correctly; frame rate is low, see below |

Tested on a Xiaomi Pad 8 Pro (Snapdragon 8 Elite, Adreno 830) running Debian 13 aarch64.

## Known issues

- **Unity IL2CPP titles do not start.** Hollow Knight (PPSA34971), Silksong (PPSA12544) and
  Biped 2 (PPSA23066) all read a global singleton that is still null. In one eboot that
  singleton has 38 reads, zero writes and its address is never taken, with relocations intact,
  so whatever populates it lives in another module.
- **Frame rate is GPU bound, not CPU bound.** Balatro runs at 9-20 FPS while the system sits at
  roughly 32% CPU (67% idle) and issues about 680 draws per frame. Synchronisation in the
  memory packet path (`WaitIdle`) and the opportunistic reap are the current suspects.

## Relationship to upstream

This is not a replacement. It tracks `boykopovar/AnyPS5` `main` and adds commits on top, so
`git log upstream/main..main` is the complete delta. The fixes are written to be upstreamable
and may be proposed there.

Read [docs/dev/BUILD-ARM64.md](docs/dev/BUILD-ARM64.md) for building on aarch64.

## License

GNU General Public License version 2 only, same as upstream.

---

# 中文

## 这是什么

[AnyPS5](https://github.com/boykopovar/AnyPS5) 的 **Linux ARM64 适配分支**。

上游的做法是把 PS5 的可执行文件重链接成**原生 x86-64 ELF**，链到重新实现的系统库上。
在 ARM64 上这个 ELF 必须经过 x86-64 翻译层运行（本项目使用 FEX），所以这里专门处理
那条路径上才会出现的问题。

## 本分支的改动

全部是「上游 main + 少量可审查的提交」。完整推理和逐项补丁见
[CHANGES-ANYPS5.md](CHANGES-ANYPS5.md) 和 `changes/` 目录。

| 修复 | 效果 |
|---|---|
| `CB_TARGET_MASK` / `CB_SHADER_MASK` 复位值改成 `0xffffffff` | GCN/RDNA 硬件上这两个寄存器复位为「所有通道启用」。AnyPS5 初始化为 0，导致每次绘制的 `hasColorTarget` 都是 false —— **依赖默认值的游戏会正常跑、有声音，但画面全黑**。 |
| 零输入的像素着色器 | 声明了 0 个插值输入却引用插值参数 0 时，原检查 `0 >= 0` 成立就判失败，整个着色器编译不过。应当把该引用解析成常量。 |
| 深度维护绘制开关 | `DB_RENDER_CONTROL` 的 depth copy / resummarize / decompress 未实现，原实现直接终止进程。加 `APS5_ALLOW_DEPTH_MAINTENANCE=1` 放行，让 2D 游戏继续运行。 |

在 Balatro（PPSA21402）上实测：修好 mask 之后，能写到色彩目标的绘制从
**387,183 次中的 0 次** 变成 **81,935 次中的 79,709 次**。

## 运行要点

- 转换产物布局是 `eboot` + `libs/` + `app0/`，游戏资源必须能从 `app0/` 访问
  （AnyPS5 从 `app0/sce_sys/param.json` 读元数据）
- `libs/libc.prx`（本项目）和 `app0/sce_module/libc.prx`（游戏自带）**两个都必须有**
- 存档挂载点是 `/savedata0`，宿主目录由 AnyPS5 创建
- 翻译层会加深调用栈，必要时 `ulimit -s` 调大

## 已验证

| 游戏 | 引擎 | 状态 |
|---|---|---|
| DOOM (2016) | id Tech 6 | 可玩。菜单 59.94 FPS，游戏内约 35 FPS |
| Balatro (PPSA21402) | LÖVE / Lua | 画面正常，帧率偏低 |

测试设备：小米平板 8 Pro（骁龙 8 Elite / Adreno 830），Debian 13 aarch64。

## 已知问题

- **Unity IL2CPP 游戏起不来**：Hollow Knight、Silksong、Biped 2 都在读取一个仍为 NULL 的
  全局单例。其中一个 eboot 里该单例有 38 处读取、0 处写入、从未取地址，重定位也完整，
  说明写入方在别的模块。
- **帧率受限于 GPU 而非 CPU**：Balatro 9-20 FPS，但系统 CPU 只用了约 32%（67% 空闲），
  每帧约 680 次绘制调用。目前怀疑是内存包路径里的同步（`WaitIdle`）和 opportunistic reap。

## 与上游的关系

本分支不替代上游，而是跟踪 `boykopovar/AnyPS5` 的 `main` 并在其上追加提交，
`git log upstream/main..main` 就是全部差异。这些修复是按可上游化的方式写的。

## 许可

GNU General Public License version 2 only，与上游一致。
