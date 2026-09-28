# tedit 插件包布局规范（v1）

*EN: tedit plugin package layout spec (v1).*

## 1. 总则

* tedit **内核为纯 tie 实现**；该铁律不约束插件开发者 —— 插件可携带库、原生
  扩展（DLL）、脚本与资产，按本规范声明与交付。
* 每个固定目录必须有**消费者**（谁加载/校验它）；无消费者的目录不设。
* 插件包以清单 `plugin.data.tie`（tie:data，tie-spec 20262 §17.1）为声明层。

*EN: the kernel is pure tie; that rule does not bind plugin developers —
plugins may ship libraries, native extensions (DLLs), scripts and assets as
declared here. Every fixed directory must have a consumer. The manifest
`plugin.data.tie` is the declarative layer.*

## 2. 目录布局（规范）

```
<plugin-repo>/
  plugin.data.tie          清单（声明层；宿主审计）
  src/
    <entry>.tie            插件入口（钩子所在；清单 "entry" 指向）
    class/                 插件携带的库（内部实现库，静态 import 编进插件）
    api/                   对外公开面（可被其他插件 import；仅此处是契约）
    extern/                外部原生依赖（DLL 等；清单 "extern" 声明）
    script/                运行时脚本（插件经 interp 自加载）
    config/                插件私有运行配置（插件自读自管）
    ui/                    UI 定义/部件（tiu 前端落地后接入）
    assets/                源资产（分发打包为 zd；打包管线下接）
```

| 目录 | 消费者 | 加载路径 |
| --- | --- | --- |
| `class/` | 插件自身 | tie 静态 import（tiec 编译期裁剪未用模块） |
| `api/` | 其他插件/宿主 | 被依赖方经静态 import 装配；依赖关系进清单 `deps`（字段待依赖装配落地时启用） |
| `extern/` | tie 运行时 | tie `extern` 声明 + 原生链接；**必须**在清单 `extern` 声明 |
| `script/` | 插件自身 | `file_read` + interp 求值（`b_eval` / `b_eval_script`） |
| `config/` | 插件自身 | 插件 `file_read` 自读；宿主不审计 |
| `ui/` | tiu 前端 | 待 tiu 终端部件规范落地后接入 |
| `assets/` | 插件打包管线 | 分发时打包为 zd（bytes/ext 载荷 + tsha1 指纹）；打包管线下接 |

*EN: layout as above — `class/` internal libraries, `api/` the only public
contract surface, `extern/` native dependencies (must be declared in the
manifest), `script/` runtime scripts self-loaded via interp, `config/`
plugin-private runtime config (host does not audit), `ui/` tiu widgets
(pending), `assets/` source assets packed to zd at distribution time.*

## 3. class 与 api 的边界

* `class/` = **内部实现库**：只许插件自己 import；其他插件不得依赖。
* `api/` = **对外契约**：其他插件依赖的唯一入口。跨插件依赖只看 `api/`，
  依赖关系在清单 `deps` 中声明（依赖装配落地前，被依赖模块由宿主装配方
  显式 import）。

*EN: `class/` is internal (never imported by other plugins); `api/` is the
only public contract surface, with cross-plugin dependencies declared in the
manifest `deps` field.*

## 4. extern：原生依赖规则

* **声明必填**：携带 DLL 的插件必须在清单 `"extern"` 声明文件名列表；
  未声明的原生文件视为非法包内容。
* **链接**：DLL 由 tie `extern` 声明对接（函数签名与符号名对应），装载时
  解析；架构随宿主（当前 x64）。
* **失败语义**：原生依赖装载失败 = **插件装载失败**，报明确诊断并跳过该
  插件；宿主不得因单个插件的 extern 失败而崩溃或拒绝启动。
* **指纹（草案字段）**：`extern_sha`（各 DLL 的 tsha1f 指纹）随原生装载
  实现启用，用于完整性校验与分发审计；启用前不作要求。
* **隔离（远期）**：原生组件的进程外隔离（tink 帧互联）为远期选项，落地
  前原生代码与宿主同进程，插件作者自负稳定性。

*EN: DLLs must be declared in the manifest `extern` list and are linked via
tie `extern` declarations; a failed native load fails only that plugin with a
clear diagnostic (the host must survive). A `extern_sha` fingerprint field is
drafted for integrity audit. Out-of-process isolation via tink frames is a
long-term option — until then native code shares the host process.*

## 5. 配置分层

* **声明层（清单）**：`plugin.data.tie` —— 宿主审计、进分发（可编译为 zd，
  语义不变）；描述插件是什么。
* **运行层（config/）**：插件私有运行配置 —— 插件自读自管，宿主不审计；
  描述插件运行时怎么变。

*EN: manifest = declarative layer (host-audited, distributed); `config/` =
runtime layer (plugin-private, not audited by the host).*

## 6. script 加载约定

* 脚本为普通 `.tie` 文件，按角色区分：`type tie<logic>`（逻辑模块，静态
  import 即可）与 `type tie<tsh>`（脚本，经 `b_eval_script` 整文件求值）。
* 加载由插件自行发起（`file_read` + interp 求值）；脚本执行上下文与插件
  同进程，脚本不得假定独立全局命名空间（interp 全局与宿主共享，前缀隔离
  由插件自负）。

*EN: scripts are ordinary `.tie` files by role (`tie<logic>` statically
imported; `tie<tsh>` evaluated via `b_eval_script`). Loading is initiated by
the plugin; interp globals are shared with the host — prefix isolation is the
plugin's responsibility.*

## 7. 分发形态：.tedit 包

* **容器**：`.tedit` = **zip 容器**（扩展名 .tedit）。v0 由内核 `src/zip.tie`
  的存储法（STORE，不压缩）写入端产出，标准 zip 读取端（解压器/资源管理器）
  直接可开；deflate 与 zip64 随载荷规模需要再补。
* **必备条目**：`plugin.zd`（清单二进制形态，`tiec --compress-data` 产出）+
  `plugin.data.tie`（清单明文形态，人类可读对账）+ `LICENSE`。载荷目录
  （assets/extern/ui 等）随插件运行时接入扩展。
* **命名草案**：`<id 的点换连字符>-<version>.tedit`（如 `example.greet` +
  `0.1.0` → `example-greet-0.1.0.tedit`）。
* **确定性**：v0 写入端固定 DOS 时间戳（2026-09-28 00:00:00）—— 同输入同包。

*EN: the distribution unit is a `.tedit` file — a zip container. v0 ships a
pure-tie STORE-method writer in the kernel (`src/zip.tie`); required entries:
`plugin.zd` + `plugin.data.tie` + `LICENSE`, payload dirs extend as the plugin
runtime lands. Naming draft: `<id with dots as dashes>-<version>.tedit`.
Fixed DOS timestamp keeps builds deterministic.*
