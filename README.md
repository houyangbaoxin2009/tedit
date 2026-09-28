# tedit

效率·性能·模块·通用

*EN: tedit — a general-purpose editor for the tie language ecosystem.*

## 目录

* [定位](#定位)
* [设计原则](#设计原则)
* [仓库版图](#仓库版图)
* [当前模块（v0 种子）](#当前模块v0-种子)
* [插件规范（v1）](#插件规范v1)
* [构建](#构建)
* [路线（ROAD）](#路线road)
* [License](#license)

## 定位

tedit 是为 tie 语言生态打造的**通用编辑器**：以能力域插件覆盖文本/代码编辑、
三维创作、图像处理、音视频剪辑、页面设计等版图（对标 vscode / blender /
unity / ps / ae / pr / dw 的能力域，不追求还原任何单一原版）。

*EN: tedit is a general-purpose editor for the tie ecosystem. Capability-domain
plugins cover text/code editing, 3D authoring, image processing, audio/video
editing and page design — modeled on the capability map of vscode / blender /
unity / ps / ae / pr / dw, not on restoring any single one of them.*

## 设计原则

* **完全模块化**：编辑器 = 内核 + 模块集合；一切能力皆模块，装配即换形态。
* **插件化**：一切非内核能力以插件交付，插件仓与核心仓彼此独立。
* **高效**：性能永远优先；交互路径零多余抽象。
* **低内存占用**：目标 4GB 内存电脑可流畅使用；按需装配、流式处理、无冗余驻留。
* **纯 tie 实现**：全量 tie 自实现，不引入 Rust 等外部实现语言。

*EN: fully modular (editor = kernel + module set); plugin-based (every
non-kernel capability ships as a plugin, in independent repos); performance
first; low memory (smooth on 4GB machines — assemble on demand, stream, keep
nothing resident needlessly); implemented purely in tie.*

## 仓库版图

本仓位于分类目录 `tedit-repo/` 下，与插件仓彼此独立：

| 仓库 | 职责 |
| --- | --- |
| `tedit`（本仓） | 内核：装配宿主 + 基础能力模块 |
| `tedit-plugin-example` | 插件规范 v1 参考实现（最小示例插件） |
| `tedit-plugin-*` | 后续能力域插件，按 ROAD 顺序逐仓落地 |

*EN: repos under the `tedit-repo/` category directory are independent: `tedit`
(kernel), `tedit-plugin-example` (reference plugin), and future
`tedit-plugin-*` capability repos.*

## 当前模块（v0 种子）

v0 从 tshell 的 tedit 终端模组子集（原 tshell-architecture §12.4）独立而来，
装配入口 `src/tedit_main.tie`：

| 模块 | 职责 |
| --- | --- |
| `lineedit` | 行编辑状态机（插入/退格/移动/历史；tie 自研，零外部 readline） |
| `complete` | 补全候选 |
| `command` | 命令引擎（分派/求值） |
| `session` | 会话加载/保存 |
| `render` | 输出渲染 |
| `sh_util` | 工具函数（token 拆分等） |
| `backend_interp` | interp 求值后端（桥接 tiec `compiler/interp`） |
| `observe` | 观测/eval-backend 配置位（动态嵌入接线点） |

## 插件规范（v1）

v1 插件为**静态装配式**（tie 静态 import 文本内联；动态加载依赖 trm 引擎
Backend 接口，接入后开放 v2 规范）。参考实现见 `tedit-plugin-example` 仓：
清单 = 插件仓根 `plugin.data.tie`（tie:data 明文，正文即 tie 表字面量，
tie-spec 20262 §17.1；分发可经 `--compress-data` 编为 zd，语义不变），钩子
约定 `plugin_on_load` / `plugin_on_unload` / `plugin_commands` + 命令处理器
`cmd_<命令名>`。注意 tie 表字面量不支持尾逗号。

*EN: v1 plugins assemble statically (tie static import); the dynamic route
awaits the trm Backend interface (v2 spec). See the `tedit-plugin-example`
repo for the reference implementation: manifest + fixed-name hooks, imported
by the host assembly.*

## 构建

需要 tiec 编译器（stage0 即可）作为本仓的**同级克隆**：`../tiec/compiler/tiec.exe`。

```tsh
tsh_main.exe -f build.tsh.tie        # 产物 src/tedit_main.exe
```

或直接：

```sh
../tiec/compiler/tiec.exe src/tedit_main.tie
```

## 路线（ROAD）

只定实现顺序，不做 forward optional：

1. **编辑器内核**：多缓冲/多视口文档模型、撤销/重做、键集分层（当前 v0）。
2. **终端前端**：tiu 终端部件持渲染与终端 UI，与引擎只走协议数据。
3. **插件运行时**：manifest 校验、钩子调度、依赖装配；随后开放 v2 动态加载。
4. **能力域插件**：文本/代码 → 页面设计 → 图像 → 音视频 → 三维/场景，
   每域独立仓 `tedit-plugin-*`。

*EN: kernel (multi-buffer document model, undo/redo, key layers) → terminal
front-end (tiu widgets own rendering; engine speaks protocol data only) →
plugin runtime (manifest validation, hook dispatch; dynamic loading later) →
capability-domain plugins, one repo per domain.*

## License

以 [Tie Public License 2.2](https://github.com/tie-lang/TPL/blob/main/tpl.txt)
（TPL 2.2）开源。
