# 工作流 Skill 绑定的身份、依赖判定与导出

决策依据：[工作流 Skill 绑定改存 Catalog ID](../../../../../decisions/desktop/plugin/category/workflow/20261126-workflow-skill-binding-by-catalog-id.md)
（desktop 范围 `implemented`）。证据固定在 desktop#847 的提交
[`ffe0b82d`](https://github.com/ora-space/desktop/commit/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71)。
下列 `Covered` 仅表示该行完整义务有直接测试断言；`Partial` 后明确保留跨入口、持久化或集成覆盖缺口。
cloud 侧适配及证据由独立的 cloud 决策和用例承接。

## Editor Must Bind New Skill Selections By Catalog ID

### 风险

编辑器如果继续写 Skill 名称，同名的本地 Skill 和插件 Skill 只能写出同一个裸名称，用户无法指定来源，启动被
`workflow_skill_ambiguous` 拒绝；之后安装一个同名插件，原本能运行的工作流也会突然被拒。反过来，如果编辑器
顺手改写已有绑定，或把 UUID 当名称展示，用户会看到无法理解的值，或在不知情时换掉了来源。

### 前置状态

- Catalog 中有本地 Skill `review` 和插件 `<namespace>/<identifier>` 提供的同名 Skill `review`，以及只有一个
  来源的 Skill `lint`。
- 一张已有工作流图的 Agent 节点绑定裸名称 `lint`，另一个节点绑定 `ns:name` 历史写法。

### 触发

1. 在 Agent 节点的 Skill 选择器中分别选择两个 `review`。
2. 打开已有工作流图，不改动其绑定后保存；再在同一节点重新选择 `lint`。
3. 在已绑定裸名称 `lint` 的节点里再添加 `lint` 的精确 Catalog ID。
4. 在同一张图里绑定两个不同 Skill，它们的调用名归一化后相同。

### 必须成立

- 选择器列出所有 Available Skill，同名条目并列并标出来源（「本地」或插件显示名）；选中后写入的
  `binding.skillId` 是 Catalog ID：本地写 UUID，插件写 `plugin:<canonical-plugin-id>:<canonical-skill-name>`。
- 已有的裸名称、`ns:name` 和插件 ID 引用在保存时不被改写；只有用户重新选择后才变成 Catalog ID。
- 绑定按解析结果显示名称和来源；解析不到时显示原始引用值并标为缺失。
- 节点内去重按解析后的 Catalog ID：裸名称唯一解析到某个 Skill 时，再添加它的精确 ID 被视为重复。
- 参与执行节点中的启用 Skill 调用名冲突在节点配置中提示，不阻止保存；备用节点不触发冲突提示。

### 禁止结果

- 同名 Skill 只能写出同一个值，或选择器只显示其中一个。
- 打开或保存图时批量把裸名称迁移为 Catalog ID。
- 把 UUID 或 `plugin:` ID 当作 Skill 名称直接显示。
- 因调用名冲突拒绝保存。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 选择器按来源区分同名 Skill，写入对应的 Catalog ID | [同名来源选择](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-inspector.test.tsx#L920) 断言插件 ID 写入；[编辑器添加并保存 Skill](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-editor.test.tsx#L1970) 断言本地 UUID 落入草稿 | `Covered` |
| 已有裸名称、`ns:name` 与插件 ID 引用在保存时原样保留，重新选择后才写入 Catalog ID | [保留 legacy:review](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-inspector.test.tsx#L920)；[保存时保留旧裸名](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-editor.test.tsx#L1970)。尚无插件 ID 引用保存、移除后重选及重新打开的完整断言 | `Partial` |
| 绑定展示按解析结果显示名称和来源，解析不到时显示原值并标缺失 | [解析后的本地名称与来源](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-inspector.test.tsx#L920)；[缺失旧引用显示原值及未安装](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-editor.test.tsx#L1970)；[运行配置显示本地名称与简介](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-run/run-act-inspector.test.tsx#L242) | `Covered` |
| 节点内按解析后的 Catalog ID 去重 | [旧引用解析到的本地 ID 不再出现在可添加菜单中](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-inspector.test.tsx#L920) | `Covered` |
| 调用名冲突只提示、不阻止保存 | [不同执行 Skill 的同调用名提示仍允许 onUpdate](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-inspector.test.tsx#L966)；[备用节点不误报](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-inspector.test.tsx#L1015)。尚无冲突图实际自动保存与重开断言 | `Partial` |

### 决策依据

- [工作流 Skill 绑定改存 Catalog ID](../../../../../decisions/desktop/plugin/category/workflow/20261126-workflow-skill-binding-by-catalog-id.md)
  D1 与不变量 1。

## Workflow Skill Dependency Status Must Match Launch Preflight

### 风险

依赖报告、运行页和启动预检如果用两套规则，报告会把启动必被拒绝的工作流显示为「已安装」，或把能启动的
工作流报成缺失：同名多候选被当作已安装、大小写不同被当作缺失、精确 ID 失效后被同名 Skill「补上」、
不可用 Skill 只给提示级警告。禁用的绑定计入横幅，又会让用户去修一个不影响启动的依赖。

### 前置状态

- 一份工作流（本地 `.orax`、市场安装、升级或重试导入）包含以下启用的绑定：存在且可用的 Catalog ID、
  已软删除记录的 Catalog ID、不对应任何记录的 `plugin:` 引用、大小写与目录不同的唯一裸名称、匹配本地和
  插件两条记录的裸名称、只匹配一条不可用记录的裸名称、只贡献一个 Skill 的插件的插件 ID 引用、贡献多个
  Skill 的插件的插件 ID 引用。
- 另有一个禁用的绑定，其引用值缺失。

### 触发

1. 将前置状态中的启用绑定拆成独立工作流样例，每个样例只启用其中一个绑定，并保留禁用的缺失绑定。
2. 对每个样例检查导入依赖报告和对应工作流快照的运行页节点配置，并对同一份图执行启动预检。若样例无法创建新运行，用已有运行夹具检查运行页。
3. 另用只含禁用缺失绑定的样例，确认明细仍显示缺失、横幅不报阻断，且启动预检不因该绑定失败。

### 必须成立

- 每个绑定按「精确 ID（含软删除）→ `plugin:` 前缀 → 插件 ID 引用 → 裸名称或 `ns:name` 的最后一段」的顺序
  解析；名称匹配 ASCII 大小写不敏感，匹配范围是全部未删除记录（含不可用）。
- 状态映射：可用记录为 `installed`；不可用记录为 `unavailable`；无记录为 `missing`；多条候选为 `ambiguous`。
  后三者都按「启动会被拒绝」呈现并计入阻断。
- 处理入口：`unavailable` 去查看 Skill；`missing` 去安装，`plugin:` 引用与插件 ID 引用按其中的插件 ID 搜索
  市场；`ambiguous` 去编辑器指定 Skill，导入预览阶段只给文字说明。
- 报告横幅只统计启用的绑定，禁用绑定照常列出状态；报告说明检查针对全局 Skill 目录。
- 对每个启用绑定，报告和运行页给出的「能否启动」与启动预检的结论一致（同一组样例同时约束前端判定与后端
  预检）。
- 「未导入」条目不参与依赖判定（继承冲突恢复决策）。

### 禁止结果

- 匹配到多条记录的裸名称或插件 ID 引用被报为 `installed`。
- 已删除的精确 ID 或 `plugin:` 引用回落到同名 Skill 并报为 `installed`。
- `unavailable` 只作为不阻断的提示。
- 禁用绑定计入阻断横幅。
- 报告与启动预检对同一绑定给出相反结论。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 前端解析顺序、大小写不敏感与「含不可用候选」规则与后端解析器一致（共享样例） | [前端共享样例](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-skill-reference.test.ts#L23) 与 [后端共享样例](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/crates/backend/src/workflow/run/prerequisites/tests.rs#L557)；[软删除 ID 不回落](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-skill-reference.test.ts#L32) 与 [删除/卸载 ID 不回落](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/crates/backend/src/workflow/run/prerequisites/tests.rs#L266) | `Covered` |
| 四种状态的映射、阻断呈现与处理入口（含按插件 ID 搜索市场） | [不可用与歧义计入阻断](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/workflow-import-report.test.ts#L393)；[缺失依赖安装入口](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/plugins-settings.test.tsx#L1418)；[不可用 Skill 设置入口](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/plugins-settings.test.tsx#L1838)；[歧义编辑器导航及保存失败保留报告](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/plugins-settings.test.tsx#L4226)；[插件 ID 搜索词](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-skill-reference.test.ts#L41)。尚无 plugin: 缺失值从报告实际跳转市场的完整断言 | `Partial` |
| 只有启用绑定计入横幅；报告说明针对全局 Skill 目录 | [禁用项保留明细而不计入阻断](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/workflow-import-report.test.ts#L363)；[真实报告横幅及全局范围说明](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/plugins-settings.test.tsx#L4203) | `Covered` |
| 本地 `.orax` 与市场安装、升级、重试共用同一判定，运行页展示使用同一结果 | [本地包报告](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/plugins-settings.test.tsx#L1255)、[市场安装](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/plugins-settings.test.tsx#L1891)、[市场升级](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/plugins-settings.test.tsx#L1982)、[卡片重试](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/plugins-settings.test.tsx#L3801)；[运行页可用本地 Skill](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-run/run-act-inspector.test.tsx#L242)。尚无所有状态逐一跨这些入口与运行页比对的集成样例 | `Partial` |

### 决策依据

- [工作流 Skill 绑定改存 Catalog ID](../../../../../decisions/desktop/plugin/category/workflow/20261126-workflow-skill-binding-by-catalog-id.md)
  D2 与不变量 2–5。
- [市场工作流冲突恢复](../../../../../decisions/desktop/plugin/category/workflow/20261124-marketplace-workflow-conflict-recovery.md)：
  「未导入」条目不参与依赖判定。

## Plugin ID References Must Resolve To The Plugin's Only Skill Or Be Ambiguous

### 风险

cloud v2 写出的工作流用 `<namespace>/<identifier>` 引用整个 Skill 插件。desktop 如果把它当裸名称，导入后
一定解析为缺失；如果在插件贡献多个 Skill 时任选一个，节点会静默用错 Skill；如果插件未安装时回落到同名
Skill，又会执行错误的说明。

### 前置状态

- 插件 A 只贡献一个 Skill，插件 B 贡献两个 Skill，插件 C 未安装；本地另有一个名称与 `<identifier>` 相同的 Skill。
- 使用独立的工作流样例，分别只启用 A、B、C 的插件 ID 引用或一个 `ns:name` 历史写法（含 `/` 和 `:`）。历史写法的名称后缀对应一个唯一可用的 Skill。

### 触发

分别对各样例检查依赖报告，并验证创建运行与切换快照的 `resume`。对能解析的样例检查冻结记录。验证 `start`、`restart` 对图引用的解析时，使用没有冻结 `catalog_skill_id` 的历史运行，并将运行置于相应命令会重新调度的状态。

### 必须成立

- 恰好含一个 `/`、不含 `:` 的引用按插件 ID 引用处理：匹配 `namespace` 等于该插件 ID 的全部未删除记录（含不可用）。
- A 的引用解析到它唯一的 Skill，冻结记录的 `catalog_skill_id` 是该 Skill 的 Catalog ID。
- B 的引用在创建运行、切换快照，以及历史运行的 `start`、`restart` 时以 `workflow_skill_ambiguous` 拒绝，候选是 B 的全部 Skill；依赖报告判为 `ambiguous`。
- C 的引用以 `workflow_skill_not_found` 拒绝，依赖报告判为 `missing`，不回落到同名本地 Skill。
- 含 `:` 的历史写法仍按最后一个冒号后的名称解析，不当作插件 ID 引用。

### 禁止结果

- 插件 ID 引用被当作裸名称匹配。
- 多 Skill 插件的插件 ID 引用静默选中其中一个。
- 插件缺失时回落到与 `<identifier>` 同名的 Skill。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 单 Skill 插件的插件 ID 引用解析并冻结为该 Skill 的 Catalog ID | [唯一插件 ID 图引用返回带 catalog_skill_id 的冻结 receipt](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/crates/backend/src/workflow/run/prerequisites/tests.rs#L446) | `Covered` |
| 多 Skill 插件的插件 ID 引用在创建运行、切换快照与历史运行的 `start`、`restart` 时以 `workflow_skill_ambiguous` 拒绝 | [共享样例](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/crates/backend/src/workflow/run/prerequisites/tests.rs#L557) 直接验证解析器拒绝 multi 插件引用；尚无该引用逐一经过创建、resume、start、restart 的直接断言 | `Partial` |
| 未安装插件的插件 ID 引用报缺失且不回落同名 | [official/lint 共享样例](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/crates/backend/src/workflow/run/prerequisites/tests.rs#L557)：本地 lint 存在仍返回缺失；[前端同一样例](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-skill-reference.test.ts#L23) | `Covered` |
| 含 `:` 的 `ns:name` 写法不被识别为插件 ID 引用 | [official/multi:lint 共享样例](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/crates/backend/src/workflow/run/prerequisites/tests.rs#L557) 与 [前端同一样例](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-skill-reference.test.ts#L23) 都解析到本地 lint | `Covered` |

### 决策依据

- [工作流 Skill 绑定改存 Catalog ID](../../../../../decisions/desktop/plugin/category/workflow/20261126-workflow-skill-binding-by-catalog-id.md)
  D4 第 5 条与不变量 3、4。
- [Skill 按存储路径以会话清单交给 Agent](../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)
  D8：冻结、预检与不回落规则。

## Uninstall Check Must Compare Skill Resolution Before And After Removal

### 风险

按「插件条目名等于引用值」判断卸载影响会同时漏报和误报：按精确 ID 或 `plugin:` 前缀绑定的引用卸载后一定
失败，却因引用值不是名称而不被列出；同名本地 Skill 仍在时，卸载插件反而解除了歧义，却被列为会失败。

### 前置状态

- 待卸载插件 P 提供 Skill `lint` 和 `review`；本地另有 Skill `lint` 和 `review`；另一插件 Q 也提供 `review`。
- 另一个待测试的插件 S 只提供一个可用的 Skill `audit`。
- 工作流草稿或已发布快照中的绑定包括：P 的 `plugin:` 引用、P 与 S 的插件 ID 引用、在本地与 P 之间歧义的裸名称
  `lint`、在本地、P、Q 之间歧义的裸名称 `review`、与 P 和 S 无关的绑定。

### 触发

从四个卸载入口之一，分别对 P 和 S 触发独立的卸载前影响检查。

### 必须成立

- 每个启用的 Skill 绑定分别以当前目录和「去掉本次待卸载插件全部 Skill 后的目录」按依赖判定规则解析。
- P 的 `plugin:` 引用和 S 的插件 ID 引用在各自的检查中，卸载前唯一解析到待卸载插件的 Skill，卸载后缺失，
  因此列为受影响并警告「启动这些工作流时会因缺少 Skill 被拒绝」。
- P 的插件 ID 引用在卸载前匹配 P 的两个 Skill、为歧义，卸载后缺失，也列为受影响：此时无法再靠指定来源解除歧义。
- 卸载 P 时，卸载前歧义、卸载后唯一解析到本地 Skill 的绑定（`lint`）不列为会失败，可作为信息提示将改用的来源。
- 卸载 P 时，卸载前后都歧义的绑定（`review`）不因本次卸载新增影响；与本次待卸载插件无关的绑定不列出。
- 上述卸载前后比较只针对启用的 Skill 绑定。禁用绑定不计入启动影响，也不会单独使其节点标题出现在警告中。只警告、不阻止，扫描范围与加载期间禁用确认等规则不变；MCP 仍按插件 ID 匹配。

### 禁止结果

- 按精确 ID 绑定 P、或按插件 ID 引用绑定 S 的工作流不被列出。
- 卸载前歧义、卸载后缺失的绑定被漏报。
- 卸载解除歧义的绑定被列为会失败。
- 警告仍写「运行到引用该插件的节点时会失败」。
- 因被引用而阻止卸载。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 精确 ID、`plugin:` 与插件 ID 引用在卸载后缺失时列为受影响 | [精确插件 ID、历史后缀与插件 ID 引用的前后解析比较](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/workflow-dependency-impact.test.ts#L513) | `Covered` |
| 卸载前歧义、卸载后缺失时列为受影响 | [multi Skill 插件 ID 引用卸载后缺失](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/workflow-dependency-impact.test.ts#L513) | `Covered` |
| 卸载解除歧义时不列为会失败；前后都歧义时不新增影响 | [本地与插件同名 lint 卸载后唯一时不列出](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/workflow-dependency-impact.test.ts#L513)。尚无卸载前后都维持歧义的专用断言 | `Partial` |
| 警告文案改为启动会被拒绝，仍只警告不阻止 | [卸载警告及确认后仍执行卸载](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/plugins-settings.test.tsx#L2191)；尚无「启动会被拒绝」完整文案的直接断言 | `Partial` |
| 禁用 Skill 绑定不计入启动影响，也不单独带出节点标题 | [禁用 Skill 不报启动影响且排除其节点标题](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/workflow-dependency-impact.test.ts#L558) | `Covered` |
| 备用非 Agent 节点的残留 Skill 配置不导致 Agent 依赖漏报 | [Agent 依赖与备用 Output 残留配置共存](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/settings/workflow-dependency-impact.test.ts#L596) | `Covered` |

### 决策依据

- [工作流 Skill 绑定改存 Catalog ID](../../../../../decisions/desktop/plugin/category/workflow/20261126-workflow-skill-binding-by-catalog-id.md)
  D3 与不变量 6。
- [市场自动导入后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261109-marketplace-workflow-import.md)
  D4：卸载入口、扫描范围与只警告不阻止。

## Workflow Export Must Replace Local Skill IDs And Keep Plugin References

### 风险

D1 之后图里存的是本机 UUID 和 `plugin:` ID。导出如果原样写出 UUID，目标机器一定解析不到；如果把插件 Skill
转回名称，目标机器有同名 Skill 时变成歧义，缺插件时也提示不出要装哪个；如果导出时改写了本机图，用户的绑定
会被悄悄换掉。

### 前置状态

- 工作流节点绑定：本地 Skill 的 UUID、已软删除本地 Skill 的 UUID、插件 Skill 的 `plugin:` ID、裸名称、
  `ns:name`、插件 ID 引用、一个本机目录里没有记录的 UUID 形式的值。

### 触发

导出该工作流，再在另一台缺少该插件、另有同名本地 Skill 的机器上导入。

### 必须成立

- 本机本地 Skill 记录（含软删除）的 UUID 导出为该记录的名称；没有对应记录的值原样导出。
- `plugin:` ID、裸名称、`ns:name` 与插件 ID 引用原样导出。
- 本机图里的绑定在导出前后不变。
- 目标机器导入后原样保存引用，`plugin:` 引用报 `missing` 并按插件 ID 提示安装，不回落到同名本地 Skill。

### 禁止结果

- 导出文件含本机 Skill 记录的 UUID。
- 插件 Skill 被导出为裸名称。
- 导出改写本机图。
- 导入时改写引用值。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 本机 Skill 的 UUID（含软删除记录）导出为名称，其他值原样导出 | [活跃/已删 UUID 与其他引用的导出矩阵](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-transfer.test.ts#L456)；[真实目录响应保留软删除名称](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/crates/backend/src/skill/package_tests.rs#L53)；[草稿与已发布导出清单、JSON 一致](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-editor.test.tsx#L2825) | `Covered` |
| 导出不改写本机图 | [导出前后完整图相等](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-transfer.test.ts#L456) | `Covered` |
| 导出再导入后插件 Skill 来源保留，缺插件时按插件 ID 提示安装 | [导出保留 plugin: 引用](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-transfer.test.ts#L456)、[安装查询使用插件 ID](https://github.com/ora-space/desktop/blob/ffe0b82df2adf2ecbbf237aaa5e1729bd3879f71/packages/app-shell/src/features/workflow-editor/workflow-skill-reference.test.ts#L41) 仅分段覆盖；尚无导出文件在另一目录状态下导入并跳转安装的完整断言 | `Partial` |

### 决策依据

- [工作流 Skill 绑定改存 Catalog ID](../../../../../decisions/desktop/plugin/category/workflow/20261126-workflow-skill-binding-by-catalog-id.md)
  D4 第 1–4 条与不变量 7、8。
