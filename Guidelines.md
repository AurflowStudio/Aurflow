# Aurflow — 核心技术栈与架构方针

> 本文档是项目的最高指导方针。所有技术决策以此为准，后续规划在此基础上展开。

---

## 技术栈

### 主语言

| 选择 | 用途 |
|------|------|
| **Rust** | 主语言。音频图引擎、节点系统、UI 渲染、脚本绑定、项目序列化 |

### 音频 I/O 与插件宿主

| 选择 | 用途 | 关系 |
|------|------|------|
| **JUCE (C++)** | ASIO / CoreAudio / ALSA 音频 I/O、MIDI、VST3 / AU / LV2 插件宿主 | 编译为静态库，与 Rust **同一进程、进程内** |

### Rust ↔ C++ 通信模型

**窄 FFI 协议（~5 个函数调用），同一进程、同一地址空间：**

```
声卡驱动 → JUCE 回调 → Rust process_graph() → JUCE 传回驱动
                ↑                              ↑
             同一个调用栈，零拷贝 float* 指针传递
```

- 非实时跨线程通信（如 UI 更新参数值）使用 **lock-free SPSC/MPMC ringbuffer + 原子变量**
- **不存在跨进程 IPC、不存在 syscall、不存在额外内存拷贝**
- 延迟与纯 JUCE C++ 方案完全一致

### 2D 渲染

| 选择 | 用途 |
|------|------|
| **Skia** (`skia-safe` crate) | 全部 2D 渲染：节点画布、旋钮/推子/按钮、波形显示、频谱 |

> 选择理由：Skia 是 Chrome / Flutter 的渲染栈，2D 矢量能力世界顶尖，文本整形（HarfBuzz）、子像素渲染、emoji 全支持。CPU 栅格化即可满足 DAW UI 需求，无需自研 GPU 渲染管线。

### 窗口与输入

| 选择 | 用途 |
|------|------|
| **winit** | 多窗口管理、鼠标/键盘事件、HiDPI 感知、IME 输入法事件 |

### 文本与输入法

| 选择 | 用途 |
|------|------|
| **cosmic-text** | 文本整形（Bidi / CJK / 连字）、IME 候选词渲染 |

> winit 已原生提供 IME 事件（Windows TSF / macOS NSTextInput / Linux ibus），cosmic-text 负责将候选词正确渲染。此组合已被 Lapce / Zed 编辑器在生产环境验证，处理中日韩输入没有障碍。

### UI 布局

| 选择 | 用途 |
|------|------|
| **taffy** | **仅**面板级布局计算（侧栏宽度、时间轴高度、混音台比例） |

> taffy **不在每帧渲染循环中执行**。仅在窗口大小改变、面板分割线拖动时触发（频率：每秒 0-2 次）。控件排布（如混音台 128 个推子）使用简单的手动 for 循环网格计算，不经过 taffy。

### UI 控件

**不使用第三方 UI 框架。基于 Skia 自绘 5 种 retained-mode 控件：**
旋钮、推子、按钮（含 toggle）、列表、树。

> 5 种控件即可覆盖 DAW 全部交互需求。基于 winit 事件做 hit-test，Skia 绘制。自绘量约 1500-2000 行 Rust。

### 脚本层

| 选择 | 用途 |
|------|------|
| **Lua / LuaJIT** (`mlua` crate) | 自动化宏、UI 面板扩展、快捷键/Action 系统、导入导出器 |

> 200KB 运行时内联，多 VM 隔离（互不阻塞），JIT 执行速度接近 C 的 1/3。已验证于 REAPER、VCV Rack。

### 自定义 DSP 节点（v2 引入）

| 选择 | 用途 |
|------|------|
| **WebAssembly** (`wasmtime`) | 用户编写自定义 DSP 算法，编译为 .wasm 后在沙箱内执行 |

> 按 buffer 粒度假调用（每次 128 sample 跨一次 wasm 边界），非逐 sample 调用。Cardinal 已验证此路径可行。

### 项目文件格式

| 格式 | 用途 | 技术 |
|------|------|------|
| **MessagePack**（主格式） | 工程文件保存/加载、预设、节点库 | `rmp-serde` — 无模式二进制，解析速度比 RON 快 10-30 倍 |
| **RON**（辅助格式） | 人类可读导出（File → Export → `*.aur.ron`）、主题、配置 | `ron` crate |

> `serde` 抽象序列化层，改一行代码切换格式。几万行自动化数据 MessagePack 加载 < 50ms。

### 插件格式支持

| 优先级 | 格式 |
|--------|------|
| 一级 | **CLAP**（MIT 许可，线性扩展架构，per-note 调制） |
| 二级 | VST3、AU、LV2 |

> 所有插件宿主能力由 JUCE C++ 侧原生管理，不在 Rust 侧重建。

### 插件 GUI 策略

- **分离浮动窗口**（v1）：双击节点弹出独立原生窗口（HWND / NSView），由 JUCE 渲染插件 GUI
- 节点画布上的"插件节点"仅显示：名称、类型图标、I/O 端口、用户选择暴露的 2-4 个宏参数
- 参数同步：VST3/CLAP 标准异步参数模型，宿主写入 → 下次 `processBlock` 生效，天然支持
- 插件 GUI 卡顿不阻塞音频线程（浮动窗口天然隔离）
- macOS 全屏模式：v1 不支持（REAPER 早期同样），v2 通过 `NSWindowCollectionBehaviorMoveToActiveSpace` 解决

### 依赖关系总图

```
Rust (主语言, Cargo 管理)
├─ skia-safe          ← 2D 渲染
├─ winit              ← 窗口 + IME 事件
├─ cosmic-text        ← 文本整形 + 输入法
├─ taffy              ← 面板布局（非每帧）
├─ mlua (LuaJIT)      ← 脚本层
├─ rmp-serde          ← MessagePack 序列化
├─ ron                ← RON 序列化
├─ wasmtime           ← WASM DSP (v2)
│
└─ JUCE (C++, 静态链接, CMake 编译)
   └─ ASIO / CoreAudio / ALSA | MIDI | CLAP / VST3 / AU / LV2
```

---

## 架构方针

### 进程模型

- **单进程**，Rust main → 静态链接 JUCE 库
- Rust ↔ C++ 在同一地址空间，音频数据零拷贝指针传递
- 跨线程通信：`crossbeam` (MPMC) / `ringbuf` (SPSC)，lock-free

### 构建系统

| 工具 | 用途 |
|------|------|
| Cargo | Rust 编译、依赖管理、构建编排 |
| CMake | 仅编译 JUCE 静态库（vendor/ 目录下） |
| `build.rs` | 自动检测已有 libjuce.a / libskia.a，若无则触发 CMake 编译 |

**构建复杂度解决方案**：
- 首次编译约 30-40 分钟（JUCE + Skia 均为大型 C++ 库，无法避免）
- 增量编译 1-3 分钟
- CI（GitHub Actions）提供预编译环境 + nightly 二进制
- Docker / DevContainer 提供一键开发环境
- 贡献者指南仅需 3 步：装 Rust → clone → `cargo build`

### 许可证

**GNU AGPL v3** — 已在仓库根目录 `LICENSE` 文件。

---

## 已识别风险与对策

| 风险 | 状态 | 对策 |
|------|------|------|
| Rust + JUCE 混合 DAW 无开源先例 | 可控 | 先写 200 行 POC 验证 FFI 音频通路，再搭全架构 |
| Skia / JUCE 首次编译耗时长 | 接受 | 文档告知 + CI 预编译 + DevContainer |
| VST3 插件 GUI 嵌入 | 规避 | v1 使用分离浮动窗口（REAPER / Bitwig 模式） |
| macOS 全屏模式下浮动窗口层级 | v2 处理 | `NSWindowCollectionBehaviorMoveToActiveSpace` |
| `skia-safe` 版本跟进上游有延迟 | 中等 | Skia API 稳定，版本锁定 + 定期更新 |
| WASM DSP 每 buffer 跨边界开销 | 低 | 按 buffer 粒度调用（非逐 sample），v2 引入，v1 不需 |

---

## 移动端迁移可行性分析

> 2026-07-24 评估结论：**技术可行，无需更换核心组件，主要挑战在产品定位而非技术栈。**

### 各组件移动端兼容性

| 组件 | iOS | Android | 备注 |
|------|-----|---------|------|
| Rust | ✅ tier 2 | ✅ tier 2 | `aarch64-apple-ios` / `aarch64-linux-android` 可用 |
| Skia | ✅ 原生 | ✅ 原生 | Chrome / Flutter / Android 自身的渲染栈 |
| winit | ✅ | ✅ | 官方支持两大移动平台 |
| cosmic-text | ✅ | ✅ | 纯 Rust，平台无关 |
| taffy | ✅ | ✅ | 纯 Rust，平台无关 |
| JUCE | ✅ CoreAudio + AUv3 | ✅ AAudio / OpenSL | 官方支持 iOS / Android 音频 I/O |
| mlua (LuaJIT) | ⚠️ 需处理 | ✅ | iOS 禁止 JIT，需回退**纯 Lua 解释模式**或 `mlua` 的无 JIT 后端 |
| wasmtime | ⚠️ 需处理 | ⚠️ | iOS 禁止 JIT，需改用 `wasmi`（纯 Rust WASM 解释器）或桌面端 AOT 预编译 |
| rmp-serde / ron | ✅ | ✅ | 纯 Rust，无平台依赖 |
| VST3 / AU / LV2 | ❌ 不存在 | ❌ 不存在 | 仅桌面端。iOS 替代为 **AUv3**（JUCE 支持宿主），Android 无标准插件格式 |

### 需要条件编译处理的两个组件

```
                桌面端               iOS                    Android
Lua             LuaJIT (JIT)        纯 Lua (解释模式)       LuaJIT (JIT)
WASM DSP        wasmtime (JIT)      wasmi (解释器)          wasmtime 或 wasmi
```

两处都只需 `#[cfg(target_os = "ios")]` 级别的条件编译切换，**不需要更换组件**。

### 产品定位约束

技术可行 ≠ 产品可行。真正瓶颈不在代码：

- **手机屏幕无法承载节点编辑器**。合理目标为 **iPad / Android 平板**（类比 FL Studio Mobile、GarageBand、Koala Sampler）
- 手机版仅适用于：**播放控制器、演出触发表面、远程混音辅助**
- 移动端插件生态远弱于桌面（仅 iOS AUv3 可用），移动版应侧重**内置节点**而非第三方插件

### 是否需要提前修改技术栈

**不需要。** 当前选择的所有核心组件（Rust、Skia、winit、JUCE）已是移动端一等公民。两个需要条件编译的点（Lua JIT、WASM JIT）是桌面端性能优化手段，去掉后移动端依然可用，且可随时加回。
