---
status: implemented
---

# 工作流 Skill 绑定改存 Catalog ID，依赖报告与卸载检查按运行时解析判定

Desktop 工作流编辑器为 Agent 节点新选择的 Skill 一律写入 Catalog ID（本地 Skill 是 UUID，插件 Skill 是
`plugin:<canonical-plugin-id>:<canonical-skill-name>`），用户因此能在同名 Skill 之间指定来源；已有图里的裸
名称不迁移，继续按运行时规则兼容。导入依赖报告、卸载影响检查和运行页展示不再按 Skill 名称精确比对，
改为与启动预检使用同一套解析规则，新增「同名歧义」状态，按阻断运行计入报告横幅。卸载检查比较卸载前后
的解析结果，只把「卸载后启动会被拒绝」的绑定列为会失败。导出把本机 Skill 的 UUID 换成名称，插件 Skill
保留 `plugin:` 形式；desktop 同时接受 cloud v2 写出的插件 ID 引用 `<namespace>/<identifier>`，cloud v2 的
workflows 契约同时接受 `plugin:` 形式（D4）。

本决策的 desktop 范围已实现并验证，状态为 `implemented`。涉及 desktop 的 `packages/app-shell` 工作流编辑器、
导入依赖报告、卸载影响检查、运行页配置展示与工作流导出，以及 `crates/backend` 工作流 Skill 解析器新增的插件 ID 引用；另要求 cloud v2 的
workflows 契约多认一种插件 Skill 写法（D4 第 6 条），cloud 侧不在本文实现范围内。后端启动预检与运行冻结规则
（[Skill 按存储路径交付](../../../core/agent/session/skill/0-skill-catalog-from-storage-paths.md) D8）是本文的判定
基准，本文只在解析表中加入插件 ID 引用一行，冻结、预检时机、不回落和调用名冲突规则不变。工作流图 JSON
结构不变，`agentConfig.skills[].skillId` 仍是字符串，取值从「Skill 名称」扩展为「Catalog ID、插件 ID 引用或
历史裸名称」。核心用例见 [`test-cases/desktop/plugin/category/workflow/workflow-skill-binding.md`](../../../../../test-cases/desktop/plugin/category/workflow/workflow-skill-binding.md)。

- **兼容性边界**：已保存的裸名称、`ns:name` 历史后缀继续可用，不做批量迁移；运行时对它们的解析和
  `workflow_skill_ambiguous` 拒绝保持不变。cloud v2 现有的 `<namespace>/<identifier>` 与裸名称含义不变。
- **不承担的迁移**：不把已有图里的裸名称改写为 Catalog ID；不改变 MCP 绑定（仍按插件 ID）；不处理 Agent
  插件的卸载检查；不改变聊天输入框 `/` 菜单的 Skill 引用；不实现 cloud v2 一侧的解析改动。

## 背景

Skill Catalog 允许同名 Skill 并存：同一名称可以同时来自本地导入和一个或多个插件，它们的 Catalog ID 不同
（[静态 Skill 包](../skill/0-static-skill-package.md) D5）。运行时已经按 Catalog ID 区分它们：启动预检和运行
冻结先按精确 Catalog ID 解析，`plugin:` 前缀不回落同名，裸名称在多个候选（含不可用候选）时以
`workflow_skill_ambiguous` 拒绝。

编辑器、依赖报告和卸载检查仍停留在名称层面：

- 编辑器的 Skill 选择器以 `skill.name` 作为选项值，同名 Skill 只能写出同一个裸名称，遇到同名时用户无法
  选择来源，运行在启动时被拒绝。
- 导入依赖报告按名称精确、区分大小写地查目录，与运行时的大小写不敏感、精确 ID 优先不一致；同名多候选被
  报成「已安装」，实际启动被拒；禁用的绑定也计入缺失横幅。
- 卸载检查按「待卸载插件拥有的 Skill 条目 `name` 等于引用值」判断（[市场工作流导入](20261109-marketplace-workflow-import.md)
  D4），已知会漏报和误报：同名的本地 Skill 仍在时，卸载插件可能反而解除歧义；按精确 ID 绑定的引用则一定失败。

工作流文档是 desktop 和 cloud v2 共用的格式，两端对插件 Skill 引用的写法不兼容：desktop 写
`plugin:<canonical-plugin-id>:<canonical-skill-name>`，指向插件内的单个 Skill，把 `<namespace>/<identifier>` 当作
裸名称而解析为缺失；cloud v2 写 `<namespace>/<identifier>`，按整个 Skill 插件下发，把含 `/` 的引用按插件身份
拆分，`plugin:<id>:<name>` 会拆出错误的身份并报未下发。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [Workflow 插件类别根决策](0-workflow-package-format.md)：文档格式、逐文档导入与校验 | 继承，不变：图结构不变，`skillId` 取值范围扩展（D1、D4） |
| [市场工作流导入](20261109-marketplace-workflow-import.md) D4：市场报告与本地 `.orax` 导入共用 `collectWorkflowDependencies` 判定 | 继承共用要求；**修改**判定规则：按运行时解析，新增歧义状态，只有启用的绑定计入横幅（D2） |
| 同上 D4：卸载检查中 Skill 按条目 `name` 与引用值精确、区分大小写匹配，不采用运行时规则 | **修改**：按卸载前后的运行时解析结果判断（D3）；MCP 仍按插件 ID，Agent 插件仍不检查 |
| 同上 D4：卸载警告「运行到引用该插件的节点时会失败」 | **修改**：Skill 依赖在启动预检时即被拒绝，警告改述为启动会被拒绝（D3） |
| 同上「本决策未解决的问题」：卸载警告与运行时一致的依赖匹配 | **修改**：Skill 部分由本文承担；Agent 插件检查仍未解决 |
| [市场工作流冲突恢复](20261124-marketplace-workflow-conflict-recovery.md)：「未导入」条目不参与依赖判定 | 继承，不变 |
| [Skill 按存储路径交付](../../../core/agent/session/skill/0-skill-catalog-from-storage-paths.md) D8 第 1、3、4、7 条：冻结、预检、精确 ID 不回落、调用名冲突 | 继承，不变：本文所有判定以它为基准 |
| 同上 D8 第 2 条：裸名称歧义拒绝；编辑器改存 Catalog ID 留待后续决定 | 歧义规则继承；**修改**：编辑器写法由本文 D1 定下，解析新增插件 ID 引用（D4 第 5 条），多 Skill 插件的插件 ID 引用同样以 `workflow_skill_ambiguous` 拒绝 |
| [静态 Skill 包](../skill/0-static-skill-package.md) D5：插件 Skill 的 Catalog 身份，`namespace` 保存完整 canonical Plugin ID | 继承，不变：插件 ID 引用按该字段匹配 |

## 决策

### D1：编辑器新选择的绑定写 Catalog ID，裸名称继续兼容

1. Skill 选择器的选项值是 Catalog ID，选中后 `binding.skillId` 写入该 ID。本地 Skill 写 UUID，插件 Skill 写
   `plugin:<canonical-plugin-id>:<canonical-skill-name>`。
2. 选择器列出所有 Available Skill，标签显示名称和来源（「本地」或插件显示名）；同名条目并列出现，靠来源区分。
3. 已有绑定原样保留：图里的裸名称、`ns:name` 和插件 ID 引用不被改写，用户重新选择该节点的 Skill 时才写入
   Catalog ID。
4. 绑定展示按 D2 的解析结果把引用值显示为 Skill 名称和来源；解析不到时显示原始引用值并标明缺失，不把
   UUID 或 `plugin:` ID 当作名称直接展示。
5. 同一节点内去重按解析后的 Catalog ID 判断：裸名称唯一解析到 X 时，再添加 X 的精确 ID 视为重复。
6. 同一张图参与执行的节点中，两个启用的不同 Skill 解析出相同调用名时，运行会在启动时被拒绝（D8 第 7 条）。
   编辑器按后端成员分析得到的执行节点提示这种冲突，不把未连接的备用节点计入；提示不阻止保存。

为什么写 Catalog ID：名称不是 Skill 的身份，只有 Catalog ID 能在同名时稳定指向用户选的那一个。之后再安装
同名 Skill，已绑定的工作流不受影响，不会突然在启动时因歧义被拒。

代价：本地 Skill 删除后重新导入会得到新的 UUID（只有包丢失、记录仍在时才沿用原 ID），原绑定解析不到，
需要用户重新选择；D2 会把它报为缺失并给出处理入口。图 JSON 的可读性下降，由 D4 在导出时处理。

### D2：依赖判定与启动预检使用同一套解析规则

导入依赖报告（本地 `.orax` 与市场安装、升级、重试共用）、运行页的节点配置展示和 D3 的卸载检查，对每个
Skill 绑定按启动预检的规则解析，按表中顺序取第一条命中的规则：

| 引用值 | 解析 |
| --- | --- |
| 等于某条 Catalog 记录的 ID（含已软删除的记录） | 只认这一条；已删除视为缺失，不回落同名 |
| 以 `plugin:` 开头但不是任何记录的 ID | 缺失，不回落同名 |
| 插件 ID 引用：恰好含一个 `/`、不含 `:`（`<namespace>/<identifier>`） | 匹配 `namespace` 等于该插件 ID 的全部未删除记录（含不可用）；0 条缺失，1 条即该记录，多条为歧义；不回落同名（D4 第 5 条） |
| 其他（裸名称，或 `ns:name` 取最后一个冒号后的部分） | 按名称 ASCII 大小写不敏感匹配全部未删除记录（含不可用）；0 条缺失，1 条即该记录，多条为歧义 |

解析结果映射为报告状态：

| 状态 | 条件 | 是否阻断启动 | 处理入口 |
| --- | --- | --- | --- |
| `installed` | 解析到一条 Available 且包可用的记录 | 否 | 无 |
| `unavailable` | 解析到一条记录，但不可用或包不可用 | 是 | 去查看 Skill |
| `missing` | 没有解析到记录 | 是 | 去安装（`plugin:` 引用和插件 ID 引用按其中的插件 ID 搜索市场） |
| `ambiguous` | 裸名称或插件 ID 引用匹配到多条记录 | 是 | 去编辑器为该节点指定 Skill；导入预览时工作流尚未创建，只给文字说明 |

- 报告横幅只统计**启用**的绑定；禁用绑定照常列出状态，但不计入阻断数，因为启动预检只检查启用的绑定。
- `unavailable`、`missing`、`ambiguous` 都按「启动会被拒绝」呈现。原来 `unavailable` 只是提示级，与启动预检
  拒绝不可用 Skill 的事实不符。
- 报告说明依赖检查针对全局 Skill 目录，不针对某个工作区。
- 前端判定和后端预检由同一组用例约束，避免两套规则再次漂移。

为什么不只修大小写：精确 ID 优先、`plugin:` 不回落和歧义都会改变「能否启动」的结论，只对齐其中一部分，
报告仍会给出与启动预检相反的答案。

### D3：卸载检查比较卸载前后的解析结果

本节修改[市场工作流导入](20261109-marketplace-workflow-import.md) D4「卸载被工作流引用的 MCP / Skill 插件」中 Skill
的身份边界，入口、扫描范围（草稿与已发布快照）、只警告不阻止、加载期间禁用确认等规则不变。

只对启用的 Skill 绑定，分别以当前目录和「去掉待卸载插件全部 Skill 后的目录」按 D2 解析：

| 卸载前 | 卸载后 | 呈现 |
| --- | --- | --- |
| 解析到待卸载插件的 Skill | `missing` | 列为受影响：卸载后启动会被拒绝 |
| `ambiguous`，候选全部属于待卸载插件 | `missing` | 列为受影响：卸载后启动会因缺少 Skill 被拒绝，无法再靠指定来源解除歧义 |
| `ambiguous`，候选含待卸载插件的 Skill | 唯一解析到其他 Skill | 不列为会失败；可作为信息提示该节点将改用哪个来源 |
| `ambiguous` | 仍为 `ambiguous` | 不因本次卸载新增影响，沿用 D2 的歧义状态 |
| 与待卸载插件无关 | 不变 | 不列出 |

警告文案从「运行到引用该插件的节点时会失败」改为「启动这些工作流时会因缺少 Skill 被拒绝」。MCP 仍按插件
ID 匹配；Agent 插件仍不在检查范围内。

为什么比较前后：同名时卸载一个插件可能反而解除歧义，单看「引用值是否等于该插件的条目名」会误报；按精确
ID 绑定插件 Skill 的引用一定失败，按名称匹配又可能因大小写或 `ns:name` 写法漏报。

### D4：导出写 `plugin:` 形式，两端互认对方的插件 Skill 写法

1. 导出不写出本机 Skill 的 UUID：引用等于本机某条本地 Skill 记录的 ID（含已软删除的记录）时，导出为该记录
   的名称。本机目录里没有对应记录的值不是本机写出的引用，原样导出。
2. 插件 Skill 的引用导出为 `plugin:<canonical-plugin-id>:<canonical-skill-name>`，与 D1 写入的值相同，不做转换。
   目标机器缺插件时，D2 按其中的插件 ID 提示要安装哪一个。
3. 裸名称、`ns:name` 和插件 ID 引用原样导出。
4. 导出只改写导出文件，不改写本机图里的绑定。本地导入、市场导入照常接受任何形式的引用并原样保存，由 D2
   判定状态。
5. desktop 的解析器（启动预检、运行冻结、D2 判定共用）接受 cloud v2 写出的插件 ID 引用
   `<namespace>/<identifier>`：插件只贡献一个 Skill 时解析到它；贡献多个时为歧义，启动以
   `workflow_skill_ambiguous` 拒绝，候选是该插件的全部 Skill；没有记录时为缺失。它不回落到同名 Skill。
6. cloud v2 的 workflows 契约多认一种写法：`ResolveWorkflowSkills` 识别 `plugin:` 前缀，取出其中的插件 ID，
   按现有的整个插件下发处理，下发粒度不变；`<namespace>/<identifier>` 和裸名称的含义不变。同一节点的多条引用
   解析到同一插件 ID 时（例如同一插件的两个 `plugin:` 引用，或 `plugin:` 引用与插件 ID 引用并存），按插件 ID
   合并为一个交付项，不重复投射同一插件。这一处属于 phase2
   的 desktop↔cloud 契约，不适用 v2 豁免，由 [ora-space/circus#1593](https://github.com/ora-space/circus/issues/1593) 在 phase2 workflows 契约中按正常 specs 流程承接（cloud ADR、
   实现与核心用例），名称段是否另作校验也由那份决策定。

为什么是 `plugin:` 形式：它与 desktop 的 Catalog ID 一致，desktop 导出再导入不丢「插件里哪个 Skill」；cloud 写出的
插件 ID 引用 desktop 照样能读，所以它包含了「导出为插件 ID 引用」方案能做到的一切。cloud 只多认一种写法，
现有写法的含义和下发粒度都不变。

## 不变量

1. 编辑器新写入的 Skill 绑定都是 Catalog ID；已有裸名称、`ns:name` 和插件 ID 引用在用户重新选择前保持原样。
2. 依赖报告、运行页展示和卸载检查对同一绑定给出的「能否启动」结论，与启动预检一致。
3. 精确 Catalog ID、`plugin:` 前缀引用和插件 ID 引用在任何判定中都不回落到同名 Skill。
4. 裸名称或插件 ID 引用匹配到多条记录（含不可用记录）时，判定结果是歧义，按阻断运行呈现。
5. 只有启用的绑定计入阻断横幅。
6. 卸载检查只把「卸载后会缺失」的启用绑定列为会失败；卸载只警告、不阻止的规则不变。
7. 导出文件不含本机 Skill 的 UUID，插件 Skill 引用保留 `plugin:` 形式。
8. D1 与 D4 在同一个 desktop 正式版中发布。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| 只在有同名时写 Catalog ID，唯一时仍写裸名称 | 多数图保持可读，导出基本不用转换 | 当时唯一的裸名称，之后安装同名插件就会在启动时被拒，用户要回编辑器重选；违反不变量 1 想要的稳定性 |
| 导出时全部转回名称 | JSON 最简单，两端都按名称解析 | 目标机器有同名 Skill 时重新变成歧义，也提示不出缺哪个插件；违反不变量 7 |
| 导出为 cloud v2 的插件 ID 引用 `<namespace>/<identifier>` | cloud 不用改 | 丢失插件内具体是哪个 Skill，多 Skill 插件在 desktop 导出再导入后退化成歧义 |
| 绑定对象增加来源字段，例如 `{ "skillId": "<name>", "plugin": "<id>" }` | 名称和来源分开，两端都能精确表达 | 两端的图结构、发布校验和编辑器都要改，可能还要升 `agentConfig` 版本，改动面最大 |
| 歧义归入 `unavailable`，配专用文案 | 改动小 | `unavailable` 的修法是检查 Skill，而歧义的修法是指定来源；原有提示级也会低估启动即拒绝的后果 |
| 批量把已有裸名称迁移成 Catalog ID | 一次性消除歧义来源 | 同名多候选时无法替用户决定来源；唯一时迁移会把「按名称」的意图固化成某个 ID，用户删除重建后反而失效 |
| 卸载检查继续按条目名称匹配，只改成大小写不敏感 | 改动小 | 精确 ID 绑定、`plugin:` 前缀和歧义解除仍会漏报或误报，违反不变量 2 |

## 风险与迁移

- **D1 不能先于 D4 发布**：D1 让新绑定写入 UUID 和 `plugin:` ID，如果导出转换没有同时发布，这期间导出的文件
  在别的机器上解析不到本地 Skill。实现可以先开发 D1–D3，但要和 D4 一起合并；或者 D4 完成之前 desktop 不发
  正式版。
- **cloud 适配之前**：desktop 导出的 `plugin:` 引用在 cloud v2 上报未下发。这不比现状更差：现在 desktop 导出的
  插件 Skill 是裸名称，cloud v2 只在个人 Skill 目录里按名称查找，同样下发不了插件。cloud 按 D4 第 6 条适配后
  才能运行。
- **本地 Skill 重建后绑定失效**：见 D1 的代价；D2 把它报为缺失并引导重新选择。
- **前后端规则漂移**：D2 要求前端判定与后端预检由同一组用例约束；实现可以由后端提供解析结果，也可以由
  前端复刻，但不能各自演进。

## 本决策未解决的问题

- 聊天输入框 `/` 菜单的同名 Skill 来源：聊天发送的是 `$<name>` 文本，没有 binding，需要另行设计引用的携带
  和解析。
- Agent 插件被工作流引用时的卸载检查。
- 删除本地 Skill 时的工作流影响提示：本文只处理插件卸载，本地 Skill 删除沿用 Skills 设置页现有行为。
- 两端市场的插件 ID 是否指向同一个插件：本文只统一引用的写法，同一插件在 desktop 与 cloud 是否得到相同的
  `<namespace>/<identifier>` 由 desktop↔cloud 市场契约保证。

## 落地顺序

1. **desktop 实现**：D1–D4 在同一版本交付（不变量 8）。保持不变：图 JSON 结构、运行冻结与预检时机、已有
   裸名称、MCP 与 Agent 插件的卸载规则。开始改变：编辑器写入 Catalog ID，解析器接受插件 ID 引用，报告新增
   歧义状态并只统计启用绑定，卸载检查按前后解析判断，导出转换本机 Skill 引用。
2. **cloud 适配**：D4 第 6 条由 [ora-space/circus#1593](https://github.com/ora-space/circus/issues/1593) 的 phase2 workflows 契约承接，与 desktop 实现互不阻塞。
3. **证据归属**：desktop 核心用例记录代表性直接证据与尚存覆盖缺口；cloud 侧状态和证据记在 cloud 的决策和
   用例里，不由本文的 `implemented` 状态代表。
