# tedit 架构（v0 种子）

*EN: tedit architecture (v0 seed).*

## 1. 定位

tedit 是为 tie 语言生态打造的通用编辑器：完全模块化、插件化、高效、低内存
占用，完全由 tie 实现。能力域版图对标 vscode / blender / unity / ps / ae /
pr / dw 的能力域，不追求还原任何单一原版。

*EN: tedit is a general-purpose editor for the tie ecosystem — fully modular,
plugin-based, efficient, low memory, implemented purely in tie. The capability
map is modeled on the domains of vscode / blender / unity / ps / ae / pr / dw,
not on restoring any single one.*

## 2. 设计原则（已定案）

* **完全模块化**：编辑器 = 内核 + 模块集合。内核只提供装配、生命周期与协议；
  一切能力（行编辑、补全、渲染、命令、文档模型、UI）皆模块，换装配即换形态。
* **插件化**：非内核能力一律插件交付；插件与内核分仓，各自独立演进、独立
  开源。内核不依赖任何具体插件；插件只依赖内核公开接口。
* **高效**：性能永远优先。交互热路径（击键→渲染）不允许经过解释性中间层。
* **低内存占用**：以 4GB 内存电脑流畅使用为验收基线。手段：按需装配（未
  import 的模块不进二进制）、流式处理大文件（不整体驻留）、会话/历史有界、
  明确内存预算（内核常驻上限随每阶段 ROAD 项一并验收）。
* **纯 tie 实现**：全量 tie 自实现，不引入 Rust 等外部实现语言（tie 生态铁律）。

*EN: kernel+module split; plugins in independent repos depending only on the
kernel's public interface; hot paths free of interpretive indirection; 4GB
RAM machines as the acceptance baseline (on-demand assembly, streaming, bounded
history, explicit memory budgets per ROAD stage); pure-tie implementation.*

## 3. 仓库版图

分类目录 `tedit-repo/` 下各仓独立（每仓自带 LICENSE 与构建）：

```
tedit-repo/
  tedit/                  内核（本仓）
  tedit-plugin-example/   插件规范 v1 参考实现
  tedit-plugin-*/         能力域插件（按 ROAD 顺序逐仓落地）
```

## 4. 当前模块地图（v0 种子）

v0 = 原 tshell tedit 终端模组子集独立（来源：tshell-architecture §12.4）。
装配入口 `src/tedit_main.tie`，当前为「会话层子集」最小宿主：

| 模块 | 层 | 职责 | 备注 |
| --- | --- | --- | --- |
| `lineedit` | L2 | 行编辑状态机 | tie 自研，零外部 readline；复杂编辑随 tty 前端深化 |
| `complete` | L2 | 补全候选 | 对缓冲 token 求候选 |
| `command` | L2 | 命令引擎 | 分派/求值；连带 observe |
| `session` | L2 | 会话加载/保存 | |
| `render` | L2 | 输出渲染 | 前端只消费协议数据 |
| `sh_util` | L1 | 工具函数 | token 拆分等 |
| `backend_interp` | L1 | interp 求值后端 | 桥接 tiec `compiler/interp`（同级克隆依赖） |
| `observe` | L2 | 求值后端配置 + 观测骨架 | tieir 观测/调试/动态加载留「接口等待」骨架 |

## 5. 插件规范 v1（已定案部分）

* **装配形态**：v1 插件为静态装配式 —— 宿主入口 `import` 插件模块（tie 静态
  import 文本内联，tiec 编译期按装配裁剪未用模块，零运行时依赖）。
* **清单（manifest）**：插件仓根 `plugin.data.tie` —— tie:data 明文形态
  （文件角色 `tie<data>`，正文即 tie 表字面量，规范基准 tie-spec 20262
  §17.1「解析由编译器背书」），字段 `id` / `name` / `version` / `license` /
  `entry`。加载路径：`file_read` 读入 → interp 求值（正文即 tie 表字面量）。
  分发时可经 `tiec --compress-data plugin.data.tie -o plugin.zd` 单向编译为
  zd 二进制（tie:zd，语义不变）。**注意：tie 表字面量不支持尾逗号**
  （`..., ]` 报 E00000）。
* **钩子**：固定命名导出 —— `plugin_on_load(ctx: i64) -> i64`（`ctx` 预留）/
  `plugin_on_unload() -> i64` / `plugin_commands() -> table<string>`（命令名
  表）+ 命令处理器约定命名 `cmd_<命令名>(args: table<string>) -> string`。
* **包布局**：插件目录布局与各目录消费者见 [plugin-layout.md](plugin-layout.md)
  —— `class/` 内部库、`api/` 对外契约、`extern/` 原生依赖（清单 `extern`
  必填声明）、`script/` 运行时脚本、`config/` 插件私有配置、`ui/`、`assets/`。
  内核纯 tie 铁律不约束插件开发者（插件可携原生扩展）。
* **v2（动态加载）**：待纯 tie 动态加载方案定案后另行开放，本版不写未定案细节。

*EN: v1 = static assembly (host imports plugin modules; unused modules trimmed
at compile time); manifest = `plugin.zd` (id/name/version/license/entry);
fixed-name hooks (`plugin_on_load` / `plugin_on_unload` / `plugin_commands`).
v2 dynamic loading is out of scope here until a pure-tie approach is settled.*

## 6. 构建

* 依赖：tiec 编译器（stage0 即可）作为本仓同级克隆 `../tiec/`。
* `tsh_main.exe -f build.tsh.tie` → `src/tedit_main.exe`；或
  `../tiec/compiler/tiec.exe src/tedit_main.tie` 直编。

## 7. 路线（ROAD）

只定实现顺序，不做 forward optional；每阶段验收含内存预算：

1. **编辑器内核**：多缓冲/多视口文档模型、撤销/重做、键集分层。
2. **终端前端**：tiu 终端部件持渲染与终端 UI，与引擎只走协议数据（对齐
   tshell 嵌入形态中的分工：引擎归 tedit，前端归部件）。
3. **插件运行时**：manifest 校验、钩子调度、依赖装配；随后开放 v2 动态加载。
4. **能力域插件**（每域独立仓，顺序即依赖序）：
   文本/代码 → 页面设计 → 图像 → 音视频 → 三维/场景。
