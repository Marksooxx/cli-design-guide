# 面向 LLM / Agent 的 CLI 规范

状态：Draft 1.0  
适用范围：新建 CLI、现有 CLI 改造、面向 AI 代码生成和团队代码评审的统一规范。

---

## 1. 目的

本规范定义一套适用于 LLM、IDE agent、自动化脚本、MCP 适配层和 CI 流程的 CLI 行为要求。

符合本规范的 CLI 必须同时满足以下目标：

- 可发现：调用方可通过 `--help` 和示例独立学会用法
- 可预测：相同输入在不同上下文中产生稳定输出
- 可恢复：错误可被纠正，危险操作有安全边界
- 可解析：输出可被稳定机读
- 可契约化：结构化输出具备版本和兼容规则

---

## 2. 规范性术语

本规范使用以下术语：

- `MUST` / `必须`：强制要求；不满足即视为不符合规范
- `MUST NOT` / `禁止`：强制禁止；出现即视为不符合规范
- `SHOULD` / `应`：强烈建议；若不满足，必须有明确、文档化的理由
- `SHOULD NOT` / `不应`：强烈不建议；若采用，必须有明确、文档化的理由
- `MAY` / `可`：可选能力

除非另有说明，所有要求均适用于：

- 顶层命令
- 子命令
- 结构化输出模式
- 非交互执行模式

---

## 3. 符合性要求

声称“符合本规范”的 CLI：

- `MUST` 满足本文所有 `MUST` 与 `MUST NOT`
- `SHOULD` 满足所有 `SHOULD` 与 `SHOULD NOT`，若未满足，必须在项目文档中记录偏离理由
- `MAY` 自主实现可选能力

团队在使用 AI 生成 CLI 时，生成结果只有在满足第 4 至第 12 节全部强制要求后，才可视为“规范合格”。

---

## 4. 命令结构

### 4.1 命令风格一致性

- CLI `MUST` 在同一工具内保持一致的命令组织风格。
- CLI `MAY` 采用资源中心型结构，例如 `mytool project create`。
- CLI `MAY` 采用任务中心型结构，例如 `mytool build` 或 `mytool get project`。
- CLI `MUST NOT` 在无明确设计理由的情况下混用多套互相冲突的命令心智模型。

### 4.2 层级深度

- 命令层级 `SHOULD` 不超过 3 层。
- 若层级超过 3 层，设计文档 `MUST` 明确说明额外层级的必要性。
- 高频命令 `MAY` 暴露顶层快捷入口，例如 `init`、`run`。

### 4.3 子命令可发现性

- 每个子命令 `MUST` 支持 `--help`。
- 顶层命令 `SHOULD` 提供可枚举的命令列表能力，例如 `commands`。

---

## 5. 参数与 Flag

### 5.1 长短 Flag

- CLI `SHOULD` 同时提供短 flag 和长 flag。
- 文档和集成说明 `SHOULD` 明确要求 agent 优先使用长 flag。
- 若某个选项仅存在长 flag，不视为违规；但其命名 `MUST` 保持清晰和稳定。

### 5.2 命名约定

除非存在充分且文档化的领域理由，下列命名 `MUST` 优先采用：

| 用途 | 规范命名 |
|---|---|
| 输出路径 | `--output` |
| 输入路径 | `--input` 或位置参数 |
| 强制执行 | `--force` |
| 跳过确认 | `--yes` / `-y` |
| 静默模式 | `--quiet` / `-q` |
| 详细日志 | `--verbose` / `-v` |
| 干跑 | `--dry-run` |
| 配置文件 | `--config PATH` |
| 非交互模式 | `--non-interactive` 或 `--no-input` |

### 5.3 输出格式 Flag

- CLI `MUST` 提供显式结构化输出开关。
- 多格式工具 `SHOULD` 使用 `--format <value>`，例如 `--format json`。
- 仅支持“默认文本 + JSON”的工具 `MAY` 使用 `--json`。
- 同一工具内的格式开关 `MUST` 保持一致。

### 5.4 布尔 Flag

- 需要显式开关的布尔选项 `SHOULD` 支持成对形式，例如 `--cache` / `--no-cache`。
- CLI `SHOULD NOT` 大量使用 `--enable-*` / `--disable-*` 作为默认模式。

### 5.5 `--verbose` 与 `--debug`

- `--verbose` `SHOULD` 表示更详细的正常运行信息。
- `--debug` `MAY` 存在，但其语义 `MUST` 与 `--verbose` 区分。
- CLI `MUST NOT` 让 `--debug` 和 `--verbose` 表达同一层含义。

---

## 6. 输出与结构化契约

### 6.1 显式结构化输出

- CLI `MUST` 支持结构化输出。
- 结构化输出 `MUST` 通过显式参数启用。
- CLI `MUST NOT` 要求调用方通过 TTY 状态推断输出格式。

### 6.2 TTY 行为

- 默认输出格式 `MUST NOT` 因 TTY 与非 TTY 环境而变化。
- TTY 检测 `MAY` 用于颜色、进度条、交互提示等展示层能力。
- TTY 检测 `MUST NOT` 用于在文本结果和 JSON 结果之间自动切换。

### 6.3 stdout / stderr 分工

- `stdout` `MUST` 承载主结果。
- `stderr` `MUST` 承载日志、进度、警告文本和辅助诊断信息。
- 在结构化输出模式下，所有机器必需的数据 `MUST` 出现在 `stdout`。
- 在结构化输出模式下，`stdout` `MUST NOT` 混入非契约化日志文本。

### 6.4 结构化结果包

快照类结构化输出 `MUST` 使用稳定的顶层包。顶层至少包含以下字段：

```json
{
  "schema_version": "1",
  "status": "ok",
  "data": {},
  "errors": [],
  "warnings": [],
  "meta": {}
}
```

字段要求如下：

- `schema_version` `MUST` 存在，类型为字符串
- `status` `MUST` 存在，取值仅允许 `ok`、`partial`、`error`
- `data` `MUST` 存在；无数据时可为 `null`、空对象或空数组，具体由命令文档定义
- `errors` `MUST` 存在，类型为数组
- `warnings` `MUST` 存在，类型为数组
- `meta` `MUST` 存在，类型为对象

### 6.5 兼容性规则

- 在同一 `schema_version` 主版本内，字段语义 `MUST NOT` 漂移。
- 新字段 `MAY` 追加。
- 已存在字段 `MUST NOT` 被静默删除或重命名。
- 不兼容变更 `MUST` 通过新的主版本 schema 暴露。
- 若提供多版本输出格式，例如 `json-v2`，切换 `MUST` 采用显式 opt-in。

### 6.6 关键字段文档化

- 所有上游自动化依赖的关键字段 `MUST` 在文档中列出。
- 列表型结果为空时，字段 `MUST` 返回空数组，`MUST NOT` 因为空而缺失。

---

## 7. 错误模型与退出码

### 7.1 文本错误

非结构化模式下，错误信息 `MUST` 至少包含：

- 错误问题
- 错误原因
- 修复建议

符合要求的示例：

```text
Error: --format value 'mp4' is not supported
Reason: 'audio convert' only accepts audio formats
Hint: try one of: wav, ogg, flac, mp3
See: mytool audio convert --help
```

### 7.2 结构化错误

结构化模式下，错误结果 `MUST` 复用第 6.4 节的顶层包。

错误项 `MUST` 至少包含以下字段：

```json
{
  "code": "INVALID_FORMAT",
  "message": "format 'mp4' not supported",
  "hint": "use one of: wav, ogg, flac, mp3"
}
```

错误项 `MAY` 追加以下字段：

- `param`
- `path`
- `retryable`
- `details`

### 7.3 拼写纠错

- 对未知子命令和未知 flag，CLI `SHOULD` 提供 `Did you mean` 提示。
- 候选数 `SHOULD` 控制在 1 到 3 个。

### 7.4 退出码

CLI `MUST` 至少稳定支持以下退出码语义：

| 退出码 | 含义 |
|---|---|
| `0` | 完整成功 |
| `1` | 执行失败或部分失败 |
| `2` | 调用方式错误 |
| `130` | 用户或宿主中断 |

补充要求：

- 参数错误、未知命令、缺失必填参数等调用问题 `MUST` 返回 `2`
- 运行时失败和部分失败 `MUST` 返回 `1`
- 发生部分失败时，退出码 `MUST NOT` 为 `0`
- 若实现更细粒度退出码，`MUST` 在文档中列出，且 `MUST NOT` 改变以上 4 个语义

---

## 8. 非交互执行与危险操作

### 8.1 非交互模式

- 所有可能触发交互提示的命令 `MUST` 支持非交互执行。
- CLI `MUST` 提供 `--yes`、`--non-interactive` 或等价能力。
- 在非交互环境下，如命令需要用户确认且调用方未显式授权，CLI `MUST` 失败并给出清晰错误。
- 上述错误 `SHOULD` 返回退出码 `2`。
- CLI `MUST NOT` 在非交互环境中静默选择默认值继续执行危险操作。

### 8.2 破坏性操作

删除、覆盖、部署、远程写入、外发、支付等操作视为破坏性或高风险操作。

此类命令：

- `MUST` 默认受保护
- `MUST NOT` 在无确认、无明确授权参数的情况下直接执行不可逆副作用
- `MAY` 通过更安全的默认语义满足要求，例如默认移入回收站、默认只生成计划

### 8.3 `--dry-run`

- 对本地文件写入、批量重命名、配置变更等可可靠预演的操作，CLI `SHOULD` 支持 `--dry-run`
- `--dry-run` 结果 `MUST` 与真实执行语义保持等价的变更视图
- CLI `MUST NOT` 提供语义不成立的假 `--dry-run`

### 8.4 `--plan`

- 对部署、基础设施、配置应用类任务，CLI `SHOULD` 优先提供 `--plan`
- `--plan` 输出 `SHOULD` 体现即将发生的变更差异

### 8.5 幂等性

- 对远程副作用命令，CLI `SHOULD` 支持幂等重试机制
- 若支持幂等，CLI `SHOULD` 提供显式参数，例如 `--idempotency-key`
- 对不支持幂等的副作用命令，文档 `MUST` 明确声明其重试风险

---

## 9. 批量任务、流式输出与部分失败

### 9.1 批量结果

对批量操作，CLI `MUST` 能表达：

- 总数
- 成功数
- 失败数
- 可定位的单项结果或失败项

批量结果 `MUST NOT` 只输出成功项而省略失败项。

### 9.2 部分失败

若批量任务中存在部分成功、部分失败：

- 顶层 `status` `MUST` 为 `partial`
- 退出码 `MUST` 为 `1`
- 结果包 `MUST` 包含失败项明细或可定位的失败引用

### 9.3 流式结构化输出

长任务、watch 模式、逐项处理流程在结构化模式下 `SHOULD` 使用 JSON Lines 或 NDJSON。

每个事件对象：

- `MUST` 包含 `event`
- `SHOULD` 包含与该事件相关的标识信息
- `MAY` 包含时间戳、进度、错误信息

推荐事件类型包括：

- `start`
- `progress`
- `item`
- `warning`
- `done`

### 9.4 流式输出的通道规则

- 流式结构化事件 `MUST` 写入 `stdout`
- 进度文本和日志 `MUST NOT` 混入结构化事件流

---

## 10. 帮助信息与内省能力

### 10.1 `--help`

每个命令的帮助页 `MUST` 至少包含：

- 用途说明
- `USAGE`
- 参数说明
- 选项说明
- 示例
- 退出码说明

示例 `MUST` 覆盖最常见的 2 到 3 个调用场景。

### 10.2 `--version`

- 顶层命令 `MUST` 支持 `--version`
- 版本输出 `SHOULD` 简洁、稳定、易于脚本读取

### 10.3 补充内省能力

以下能力非强制，但 `SHOULD` 或 `MAY` 视工具复杂度实现：

- `commands`：枚举可用子命令
- `config show`：查看当前生效配置
- `doctor`：环境自检
- `schema <cmd>`：导出命令的结构定义

---

## 11. 配置、环境变量与状态

### 11.1 配置优先级

CLI `SHOULD` 使用以下优先级：

```text
CLI flag > 环境变量 > 项目配置 > 用户配置 > 默认值
```

### 11.2 环境变量

- 环境变量 `SHOULD` 使用统一前缀，例如 `MYTOOL_`
- 凭证、密钥、endpoint、日志级别等关键配置 `SHOULD` 支持环境变量注入

### 11.3 配置位置

CLI `MUST` 在文档中明确配置文件位置和覆盖规则。推荐位置如下：

| 平台 | 推荐位置 |
|---|---|
| Linux | `~/.config/mytool/config.toml` |
| macOS | `~/Library/Application Support/mytool/` 或一致的 XDG 风格目录 |
| Windows | `%APPDATA%\mytool\config.toml` |
| 项目级 | `./mytool.toml` |

### 11.4 隐式状态

- CLI `MUST NOT` 依赖未文档化的隐藏状态改变命令主语义
- 若存在缓存、会话、最近使用对象等隐式状态，其影响范围 `MUST` 被文档化

---

## 12. 性能要求

### 12.1 启动行为

- `--help` 与 `--version` `SHOULD` 避免加载重型依赖
- 参数解析前 `SHOULD NOT` 执行与当前命令无关的昂贵初始化

### 12.2 性能目标

以下为推荐目标，而非强制上限：

- `--help` 和 `--version` `SHOULD` 明显快于业务命令
- 常见元信息命令冷启动 `SHOULD` 尽量控制在 300ms 量级

### 12.3 热路径优化

若某类命令需要重初始化，CLI `MAY` 提供：

- `warmup`
- daemon 模式
- 预热缓存

---

## 13. 禁止模式

以下模式属于规范禁止或强烈不建议行为：

- 输出格式因 TTY 状态自动切换
- 在结构化模式下向 `stdout` 混入日志文本
- 错误信息仅输出无上下文的 `Error`
- 非交互环境中静默选择默认值继续执行
- 批量任务只输出成功项
- 发生部分失败却返回 `0`
- 提供语义不成立的假 `--dry-run`
- 结构化输出无版本字段
- 在同一 schema 主版本内静默删除、重命名或改变字段语义
- 依赖未文档化的隐藏状态改变主行为

---

## 14. 最小合规清单

实现方在交付前 `MUST` 逐项确认以下内容：

- [ ] 命令结构风格一致
- [ ] 每个命令支持 `--help`
- [ ] 顶层支持 `--version`
- [ ] 支持显式结构化输出
- [ ] 默认输出格式不随 TTY 改变
- [ ] `stdout` 和 `stderr` 分工明确
- [ ] 结构化输出包含 `schema_version`、`status`、`data`、`errors`、`warnings`、`meta`
- [ ] 错误对象至少包含 `code`、`message`、`hint`
- [ ] 退出码实现 `0`、`1`、`2`、`130` 的稳定语义
- [ ] 非交互环境可执行，不会卡死在交互提示
- [ ] 破坏性操作默认受保护
- [ ] 批量任务可表达部分失败
- [ ] 文档明确配置位置、配置优先级和关键字段

---

## 15. 供 AI 代码生成使用的执行规则

当本规范作为 AI 的输入规范时，调用方应视以下规则为默认执行约束：

- AI `MUST` 以本规范为 CLI 行为的最高优先级约束
- AI `MUST` 在生成代码时同时生成 `--help`、结构化输出、错误模型和非交互模式
- AI `MUST NOT` 仅实现“能跑通 happy path”的文本输出 CLI
- AI `MUST` 在生成批量命令时实现部分失败语义
- AI `MUST` 在生成破坏性命令时实现确认保护与非交互失败策略
- AI `SHOULD` 在生成结果结构时直接采用第 6.4 节顶层包
- AI `SHOULD` 在生成评审清单时使用第 14 节作为验收标准

---

## 16. 结论

本规范的核心要求只有一句话：

> CLI 不只是终端工具；它是需要被人类和 agent 同时稳定消费的接口。

凡是会让人类困惑、让脚本脆弱、让 agent 无法纠错的设计，都不应进入一个面向自动化时代的 CLI。
