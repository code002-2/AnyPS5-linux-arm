# 我们对 AnyPS5 的改动

基线：`/tmp/aps5full` 源码树（平板与 WSL 同一份）。
下面按「功能性修复」和「诊断辅助」分开，前者改变行为，后者只加日志/开关。

---

## 功能性修复

### 1. `CB_TARGET_MASK` / `CB_SHADER_MASK` 的复位值（**最关键**）

文件：`core/libs/prx/libSceAgcDriver/Execution/include/QueueState.hpp`

```diff
-        {0x80, 0}, {0x83, 0xffff}, {0x8c, 0xaa99aaaa}, {0x8d, 0}, {0x8e, 0}, {0x8f, 0},
+        {0x80, 0}, {0x83, 0xffff}, {0x8c, 0xaa99aaaa}, {0x8d, 0}, {0x8e, 0xffffffff}, {0x8f, 0xffffffff},
```

`0x8e` = `CB_TARGET_MASK`，`0x8f` = `CB_SHADER_MASK`。**在真实 AMD GCN/RDNA 硬件上这两个
寄存器的复位值都是 `0xffffffff`**（8 个渲染目标 × RGBA 全通道启用）。

`State.cpp` 用它们判定 `hasColorTarget`：

```cpp
for (slot in 0..8) if (((shaderMask >> (4*slot)) & 0xf) != 0) exportSlots.push_back(slot);
uint32_t exportCount = 0;
for (i in 0..exportSlots.size()) if (written(exportSlots[i])) exportCount = i + 1;
if (((read(cx, 0x202) >> 4) & 7) == 0) exportCount = 0;
result.hasColorTarget = exportCount != 0;
```

两个 mask 为 0 → `exportSlots` 空 → `exportCount = 0` → `hasColorTarget = false` →
**每次绘制都写不到色彩表面，画面全黑**。

依赖硬件默认值、从不显式写这两个寄存器的游戏（Balatro）因此完全黑屏。
DOOM 不受影响，因为它会显式设置。

**验证**（Balatro）：

| | 修复前 | 修复后 |
|---|---|---|
| `colorTarget=1` 的绘制 | 0 / 387,183 | **79,709 / 81,935（97.4%）** |
| 画面 | 全黑（有音乐） | 正常 |

### 2. `sceKernelRaiseException` 的空句柄

文件：`core/libs/prx/libkernel/System/src/Exception.cpp`

```diff
 bool Send(Pthread thread) {
+    if (thread->hostThread == 0) return false;
     const int result = pthread_kill(thread->hostThread, HostSignal);
```

原代码检查了 `thread == nullptr` 和 `_finished`，但没检查 `hostThread`。
glibc 的 `pthread_kill` 没有 null 检查，句柄为 0 时崩在 `0x904`。
（这两个断言只在 `hostThread` 为 0 时自相矛盾，说明该状态确实可达。）

### 3. 深度维护绘制的放行开关

文件：`core/libs/prx/libSceAgcDriver/Graphics/src/State.cpp`

```diff
 std::string DepthMaintenanceRejection(const QueueState& queue) {
     if (DepthResummarizeDrawsNothing(queue)) return {};
+    if (std::getenv("APS5_ALLOW_DEPTH_MAINTENANCE") != nullptr) return {};
```

`DB_RENDER_CONTROL` 的 depth copy / resummarize / decompress 绘制未实现，原实现直接抛异常
终止进程：

```
what(): DB_RENDER_CONTROL depth copy, resummarize or decompress is unsupported
```

2D 游戏仍会提交这类绘制，但它们对输出没有影响。加环境变量开关放行。

### 4. 零输入像素着色器的插值引用

文件：`core/shader/recompiler/Optimization/src/ShaderInfoCollector.cpp`

```diff
-                    if (input->ImmediateU32() >= inputInfo.pixel->inputNum || component->ImmediateU32() >= 4u || mode->ImmediateU32() >= 3u) {
+                    if (inputInfo.pixel->inputNum != 0 && (input->ImmediateU32() >= inputInfo.pixel->inputNum || component->ImmediateU32() >= 4u || mode->ImmediateU32() >= 3u)) {
```

像素着色器声明了 0 个插值输入，却仍然引用插值参数 0。原检查 `0 >= 0` 成立 → 失败：

```
shader info collection failed: interpolation parameter reference is out of range
(attribute 0 component 0 mode 0, 0 inputs)
```

没有输入可读，引用只能解析成常量，不应终止整个帧。

---

## 诊断/实验性改动

这些不改变正确行为，只增加可观测性或开关，可以按需保留或丢弃。

| 文件 | 内容 |
|---|---|
| `libkernel/Pthread/src/Thread.cpp` | `DEFAULT_STACK_SIZE` 1 MiB → 8 MiB；`RunThread` 打印启动结构、host `backtrace`；`ClearGuestSingleThreadedFlag()` 目前是空操作（见下） |
| `libkernel/Pthread/src/Attr.cpp` | 同上，栈大小 8 MiB |
| `libkernel/Module/src/DynamicLoader.cpp` | `dlopen` 后打印 IL2CPP 模块基址与全局值 |
| `libSceAgcDriver/Graphics/src/State.cpp` | `hasColorTarget` 为 0 时打印 `CB_COLOR_CONTROL`/`CB_SHADER_MASK`/`CB_TARGET_MASK`（前 6 次） |
| `relinker/elfpatcher/src/linux/LinuxElfPatcher.cpp` | 尝试保留 `DT_INIT`/`DT_FINI`（**实测无效**，因为 `DynamicSegmentData` 里只有 `DT_NEEDED`；真正该改的是 `SysVDynamicSectionBuilder`） |

### 关于 `ClearGuestSingleThreadedFlag()`

曾经用来把 guest libc 的 `__libc_single_threaded` 清零，但**地址 `libc_base + 0x1988e8` 是按
Silksong 的 libc 量出来的**。Balatro 的 libc 只有 1.2 MB，该偏移没有映射 → `SEGV_ACCERR`。

而且实测该标志本来就是 0，这个探测没有价值。**现在函数体第一行就 `return`，整个是空操作。**
应当保留这个禁用状态，或直接删除该函数。

---

## 已知未解决

- **Unity IL2CPP 游戏**（Hollow Knight PPSA34971、Silksong PPSA12544、Biped 2 PPSA23066）
  在 eboot 里读取一个全局单例时拿到 NULL。该全局 vaddr `0x22F3ED0`（Silksong）在 eboot 里
  **只有 38 处读取、0 处写入、0 处取地址**，重定位也完整，说明写入方在别的模块。
- **性能**：Balatro 进游戏约 9-20 FPS。系统 CPU 只用了约 32%（67% 空闲），不是 CPU 受限；
  每帧约 680 次绘制，怀疑是 GPU 侧或同步开销。
