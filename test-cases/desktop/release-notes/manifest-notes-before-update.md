# 更新前的清单日志核心用例

依据[更新清单携带最近 10 个版本的日志，用户在确认安装之前看到](../../../decisions/desktop/release-notes/20261124-manifest-notes-before-update.md)。
下文的 D 编号和不变量编号都指这篇 ADR，指根决策时另外写明。

ADR 为 `implemented`，证据固定在 desktop 合并提交
[`a7fb0ff1`](https://github.com/ora-space/desktop/commit/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36)，路径相对 desktop 仓库根目录，行号指测试
函数、`it` 或 `Deno.test` 的起始行。30 项义务中 18 项 `Covered`、
7 项 `Partial`、5 项 `Missing`；6 个用例中 1 个 `Covered`、5 个 `Partial`。最后一个用例记的是更新入口的失败提示，
不来自本 ADR 的不变量。

| 不变量 | 用例 | 状态 |
|---|---|---|
| 2、3、5 | [The Manifest Carries The Latest Ten Versions](#the-manifest-carries-the-latest-ten-versions) | Partial |
| 4、7 | [Only The Versions An Update Brings Are Shown](#only-the-versions-an-update-brings-are-shown) | Covered |
| 5、6 | [Unusable Manifest Notes Are Hidden As A Whole](#unusable-manifest-notes-are-hidden-as-a-whole) | Partial |
| 1 | [Manifest Notes Never Change An Update](#manifest-notes-never-change-an-update) | Partial |
| 8 | [Installing Needs One Explicit Confirmation](#installing-needs-one-explicit-confirmation) | Partial |
| — | [Update Failures Show Their Reason And Can Be Retried](#update-failures-show-their-reason-and-can-be-retried) | Partial |

## The Manifest Carries The Latest Ten Versions

### 风险

清单漏掉目标版本，用户安装前看不到要装的版本改了什么；标准字段类型变化，所有客户端都检查不到更新；文件里以后加的字段泄漏到公开清单。

### 前置状态

一个带日志的 desktop 检出和已签名的安装包；日志多于 10 个版本，含高于目标版本的；分别制造目标版本没有日志、某个历史文件不合格。

### 触发

按正式版 tag 和预发布 tag 生成更新清单。

### 必须成立

- 带不高于目标正式版本的最近 10 个版本，必须包含目标版本；预发布 tag 按它的正式版本取。
- `ora_release_notes` 带 `schema` 和 `versions`，只含已知字段；标准 `notes` 是字符串。
- 目标版本没有日志或任一文件不合格时不写清单，`release` job 失败。
- 某个版本超过 50 条时打包检查失败。

### 禁止结果

- 静默产出没有日志或缺少目标版本的清单。
- 已有的清单被失败的生成覆盖。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 版本选择、字段投影和标准 `notes` | [`scripts/create-desktop-update-manifest.test.ts:39`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/scripts/create-desktop-update-manifest.test.ts#L39) `updater manifest contains complete signed targets…`：整份清单逐字段相等，日志只有已知字段，`notes` 为 `Ora <tag>`；[`scripts/create-desktop-update-manifest.test.ts:116`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/scripts/create-desktop-update-manifest.test.ts#L116) `updater manifest carries the latest ten versions…`：12 个版本、预发布 tag，取不高于目标正式版本的 10 个 | `Covered` |
| 目标版本没有日志或文件不合格时不写清单 | [`scripts/create-desktop-update-manifest.test.ts:167`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/scripts/create-desktop-update-manifest.test.ts#L167) `no updater manifest is written when …`：目标版本没有日志、文字含孤立代理项、历史文件损坏，都失败且没有写出 `latest.json` | `Covered` |
| 超过 50 条时带 `--tag` 的打包失败 | [`scripts/release-notes.test.ts:186`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/scripts/release-notes.test.ts#L186) `the shared case with too many entries is rejected`：51 条被打包检查使用的解析拒绝。没有用 51 条的文件直接跑 `package-desktop.ts --tag` 的测试 | `Partial` |
| 清单条目与同一检出的内置日志逐字相同 | [`scripts/create-desktop-update-manifest.test.ts:116`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/scripts/create-desktop-update-manifest.test.ts#L116) `updater manifest carries the latest ten versions…`：清单条目与写入的文件逐字段相等。没有同时取内置日志对比的测试 | `Partial` |
| 生成端与客户端在同一 `schema` 下的版本数、条数、码点上限和分类集合一致 | [`scripts/create-desktop-update-manifest.test.ts:186`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/scripts/create-desktop-update-manifest.test.ts#L186) `updater manifest matches the fixture the Desktop host reads` 产出的清单由 [`apps/desktop/src-tauri/src/update/release_notes.rs:293`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L293) `reads_the_manifest_the_generator_writes` 解析，并断言多一个版本就被拒绝；[`scripts/release-notes.test.ts:186`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/scripts/release-notes.test.ts#L186) `the shared case with …` 与 [`apps/desktop/src-tauri/src/update/release_notes.rs:249`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L249) `enforces_the_content_rules_of_the_notes_files` 读同一组样例：50/51 条、500/501 码点、四种分类与未知分类 | `Covered` |
| 真实 `release` job 失败与成功 | 尚无，没有实测 | `Missing` |

### 决策依据

ADR D1，不变量 2、3、5。

## Only The Versions An Update Brings Are Shown

### 风险

展示已经装了的版本，或展示与要安装的版本不符的日志，用户据此作出错误的决定。

### 前置状态

清单有多个版本；当前和目标分别是正式版、预发布、带 `+build`。另有一次成功的检查之后检查失败、保留旧的 `Ready`。

### 触发

检查更新，打开设置的更新日志页。

### 必须成立

- 展示 `(stable(当前), stable(目标)]`，按版本倒序，按界面语言。
- 日志来自产生当前 `Ready` / `ManualUpdate` 状态的那一次检查；保留旧 `Ready` 时日志仍是那个版本的。
- 日志只在“可更新”区块里，历史列表不变。
- 展示清单日志不写入“已提示过的最高版本”，更新后的卡片照弹。

### 禁止结果

- 清单日志进入历史列表，或被持久化。
- 前端拿到原始清单。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 区间与预发布、`+build` 的归一化 | [`apps/desktop/src-tauri/src/update/release_notes.rs:211`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L211) `selects_the_versions_an_update_would_bring` | `Covered` |
| 日志与更新状态绑定，保留 `Ready` 时不换 | [`apps/desktop/src-tauri/src/update/tests.rs:362`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/tests.rs#L362) `offers_carry_the_notes_of_their_own_manifest`、[`apps/desktop/src-tauri/src/update/tests.rs:410`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/tests.rs#L410) `a_failed_check_keeps_the_ready_release_and_its_notes`：规则有测试；[`apps/desktop/src-tauri/src/update/engine_tests.rs:166`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/engine_tests.rs#L166) `offered_notes_start_after_the_running_package_version`：区间下界取运行中安装包的版本；[`apps/desktop/src-tauri/src/update/engine_tests.rs:187`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/engine_tests.rs#L187) `each_offer_carries_the_notes_of_its_own_check`：连续两次检查各带自己清单的日志；[`apps/desktop/src-tauri/src/update/engine_tests.rs:225`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/engine_tests.rs#L225) `a_failed_check_keeps_a_ready_release_only_while_its_package_verifies`：检查失败时保留的包仍通过校验才保留 `Ready` 和日志，否则报告检查失败 | `Covered` |
| 清单日志不进入历史列表、不影响已提示记录 | [`packages/app-shell/src/features/settings/settings-dialog-release-notes.test.tsx:80`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/settings/settings-dialog-release-notes.test.tsx#L80) `opens the offered release above a history that does not list it`：`ready` 与 `manual_update` 各一遍，断言区块在历史之上、历史不含该版本、已提示记录不变 | `Covered` |
| 清单日志不被持久化 | [`apps/desktop/src-tauri/src/update/engine_tests.rs:327`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/engine_tests.rs#L327) `manifest_notes_are_never_persisted`：更新缓存里没有日志文字，下一个进程从缓存恢复同一个包时只显示新清单的日志，且不重新下载；前端不写已提示记录由上一行的 `settings-dialog-release-notes.test.tsx:80` 断言 | `Covered` |
| 按界面语言展示 | [`packages/app-shell/src/features/workspace/pending-update.test.tsx:64`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L64) `shows the offered notes in the interface language`：en-US 下“可更新”区块显示英文条目，不显示中文 | `Covered` |

### 决策依据

ADR D3、D5，不变量 4、7。

## Unusable Manifest Notes Are Hidden As A Whole

### 风险

清单没有签名。接受超限或不合规则的内容会冲击界面或误导用户；只丢掉一部分，用户会以为看到的是全部。

### 前置状态

清单的日志分别：超过 10 个版本、某版超过 50 条、文字超过 500 码点、分类未知、缺一种语言、含控制或双向控制字符、同一版本出现多次、日期不存在、`schema` 未知；另有带未知字段、没有日志字段的清单。

以上都是更新器能成功解析的清单。含孤立的 UTF-16 代理项的清单会被更新器整份拒绝，读取日志之前就失败，所以由生成端拦截：这样的日志文件
被拒绝，不写出清单。

### 触发

检查更新后读取日志。

### 必须成立

- 任一不合规则的情形都隐藏全部日志，并在 Desktop 日志里记录原因。
- 未知字段被忽略；没有日志字段不算异常。
- 打包检查与客户端对同一组样例判断一致。
- 日志按纯文本显示，不产生元素、样式或链接。

### 禁止结果

- 只展示剩下的合格部分，或截断文字。
- 前后两端对同一条内容判断不同。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 每种不合规则的情形都整组隐藏 | [`apps/desktop/src-tauri/src/update/release_notes.rs:269`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L269) `hides_all_notes_when_any_of_them_is_unusable`、[`apps/desktop/src-tauri/src/update/release_notes.rs:249`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L249) `enforces_the_content_rules_of_the_notes_files`、[`apps/desktop/src-tauri/src/update/release_notes.rs:320`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L320) `hides_notes_that_repeat_a_version` | `Covered` |
| 未知字段忽略、未知 `schema` 隐藏 | [`apps/desktop/src-tauri/src/update/release_notes.rs:353`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L353) `ignores_unknown_fields_and_hides_unknown_schemas` | `Covered` |
| 打包检查与客户端共用样例 | [`scripts/release-notes.test.ts:186`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/scripts/release-notes.test.ts#L186) `the shared case with …` 与 [`apps/desktop/src-tauri/src/update/release_notes.rs:249`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L249) `enforces_the_content_rules_of_the_notes_files` 读同一份样例，包括合法代理对、只有 U+FEFF 和 `+build`；孤立代理项在客户端一侧是 JSON 解析失败 | `Covered` |
| 含孤立代理项的日志在生成端被拒绝，不写出清单 | [`scripts/create-desktop-update-manifest.test.ts:167`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/scripts/create-desktop-update-manifest.test.ts#L167) `no updater manifest is written when a text holds a lone surrogate` | `Covered` |
| “可更新”区块只按纯文本 | [`packages/app-shell/src/features/workspace/pending-update.test.tsx:91`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L91) `shows the notes as plain text before the user installs` | `Covered` |
| 隐藏的原因写进 Desktop 日志 | 没有断言日志内容的测试 | `Missing` |

### 决策依据

ADR D2，不变量 5、6；根决策不变量 6、8。

## Manifest Notes Never Change An Update

### 风险

日志是附带内容。它缺失、损坏或超限时若改变更新结果，用户会因为一段说明文字装不上修复版本。

### 前置状态

更新可用；清单的日志分别为合格、缺失、损坏、超限。

### 触发

检查、下载、安装。

### 必须成立

- 更新器成功解析清单之后，检查、下载、安装的结果与没有日志时相同。
- 没有日志时“可更新”区块仍显示可更新的版本和安装操作。

### 禁止结果

- 日志问题变成更新错误，或使安装操作不可用。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 日志的各种情形都不改变更新状态 | [`apps/desktop/src-tauri/src/update/tests.rs:389`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/tests.rs#L389) `broken_notes_never_withhold_an_offer`、[`apps/desktop/src-tauri/src/update/release_notes.rs:333`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L333) `unusable_notes_are_hidden_instead_of_failing`、[`apps/desktop/src-tauri/src/update/release_notes.rs:370`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/release_notes.rs#L370) `accepts_manifests_without_notes`。引擎已有更新器替身，但还没有用缺失、损坏或超限的日志走完检查、下载到安装的用例 | `Partial` |
| 没有日志时仍能安装 | [`packages/app-shell/src/features/workspace/pending-update.test.tsx:169`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L169) `still offers the installation without notes`：没有日志时仍有安装操作；确认到安装的流程由带日志的测试覆盖 | `Partial` |

### 决策依据

ADR D2、D3，不变量 1；根决策不变量 7。

## Installing Needs One Explicit Confirmation

### 风险

安装会重启应用，而更新模块不检查有没有正在运行的会话。误点一次就重启会丢掉用户正在做的事。

### 前置状态

有可安装的更新；另有只能手动更新的安装。

### 触发

点更新入口、点“立即更新”、在确认框里取消或确认。

### 必须成立

- 更新入口只切到更新日志页，不安装。
- 点“立即更新”弹出不带日志的确认框，写明会自动重启；确认后才安装，取消不安装；确认之后、安装之前换成别的版本时拒绝安装，新版本要重新确认；这次拒绝是冲突（`desktop_update_release_replaced`，记 WARN），外壳不提示安装失败。
- 手动更新的安装能看日志，有固定文案，没有安装按钮。

### 禁止结果

- 没有确认就安装。
- 手动更新的安装出现安装操作，或使用清单里的链接。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 确认前不安装，取消不安装 | [`packages/app-shell/src/features/workspace/pending-update.test.tsx:91`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L91) `shows the notes as plain text before the user installs`、[`packages/app-shell/src/features/workspace/pending-update.test.tsx:115`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L115) `installs nothing when the confirmation is cancelled`、[`packages/app-shell/src/features/workspace/desktop-update-control.test.tsx:56`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/desktop-update-control.test.tsx#L56) `leads to the offered release without installing it` | `Covered` |
| 确认只对看到的版本有效，被换掉时不安装 | [`apps/desktop/src-tauri/src/update/tests.rs:339`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/tests.rs#L339) `a_replaced_release_is_not_installed_by_an_older_confirmation`、[`apps/desktop/web/tauri-platform-adapter.test.ts:46`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/web/tauri-platform-adapter.test.ts#L46) `installs the confirmed version only`、[`packages/app-shell/src/features/workspace/pending-update.test.tsx:131`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L131) `asks again for a release that replaced the confirmed one`、[`packages/app-shell/src/features/workspace/pending-update.test.tsx:154`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L154) `closes the confirmation when the release changes in one step`：前端、适配器和宿主状态各有测试；[`apps/desktop/src-tauri/src/update/engine_tests.rs:261`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/engine_tests.rs#L261) `installation_is_for_the_confirmed_version_only`：引擎只安装确认的版本，版本不符时不调用安装器；[`apps/desktop/src-tauri/src/update/tests.rs:551`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/tests.rs#L551) `a_replaced_release_is_refused_as_a_conflict`：命令以 `desktop_update_release_replaced` 拒绝、请求日志为 WARN；[`apps/desktop/web/tauri-platform-adapter.test.ts:57`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/web/tauri-platform-adapter.test.ts#L57) `rejects an installation with the decoded contract error`、[`packages/app-shell/src/features/workspace/pending-update.test.tsx:302`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L302) `does not report a confirmation that a newer release replaced`：外壳不提示安装失败并显示新版本。Tauri 命令 `install_desktop_update` 到 `UpdateService::install` 的转发没有经过命令入口的测试 | `Partial` |
| 手动更新没有安装操作 | [`packages/app-shell/src/features/workspace/pending-update.test.tsx:176`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L176) `explains a manual update without an install action` | `Covered` |
| 真实应用里确认后安装并重启 | 尚无，没有实测 | `Missing` |

### 决策依据

ADR D3、D4，不变量 8。

## Update Failures Show Their Reason And Can Be Retried

### 风险

检查或安装失败时只在禁用按钮的悬停提示里有一句技术文字，用户既不知道失败了，也不知道怎么重试；把安装失败说成检查失败，用户会去查网络而不是重试安装。

### 前置状态

发布构建；分别制造检查失败（清单取不到、下载或校验失败）、确认后安装前的最终校验失败（包被丢弃）、安装器拒绝（包保留）。另有失败之后重新挂载设置页、`checking` 与紧随的失败在同一次渲染里到达。

### 触发

打开设置，看侧栏的版本按钮和更新日志页，点“重试”或再次确认安装。

### 必须成立

- 宿主的 `Failed` 写明失败的是检查还是安装（安装带确认的版本）；侧栏、悬停提示和更新日志页按它写标题，不靠前端自己的记录推断。
- 失败时侧栏显示失败，版本按钮可以点，跳到更新日志页；不显示表示有新版本的红点。
- 更新日志页显示原因并可以重试；安装器拒绝后可以再次确认安装。
- 失败原因带完整的原因链，写进 Desktop 日志。

### 禁止结果

- 安装失败被写成检查失败，或在重新挂载、合并渲染后变成另一种失败。
- 失败时仍显示红点，或版本按钮不可点、没有重试的入口。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| `Failed` 带失败阶段，检查失败和最终校验失败各报告对的阶段 | [`apps/desktop/src-tauri/src/update/tests.rs:470`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/tests.rs#L470) `status_serializes_to_the_shape_the_webview_consumes`：两种阶段的线上形状；[`apps/desktop/src-tauri/src/update/engine_tests.rs:225`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/engine_tests.rs#L225) `a_failed_check_keeps_a_ready_release_only_while_its_package_verifies`：检查失败报告 `check`；[`apps/desktop/src-tauri/src/update/engine_tests.rs:296`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/apps/desktop/src-tauri/src/update/engine_tests.rs#L296) `a_package_that_fails_its_last_verification_is_an_install_failure`：最终校验失败报告 `install` 和版本，且不调用安装器 | `Covered` |
| 侧栏和更新日志页按阶段写标题，重新挂载和合并渲染后不变 | [`packages/app-shell/src/features/workspace/desktop-update-control.test.tsx:96`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/desktop-update-control.test.tsx#L96) `shows a failed check and leads to its reason without advertising a release`、[`packages/app-shell/src/features/workspace/desktop-update-control.test.tsx:115`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/desktop-update-control.test.tsx#L115) `does not call a failed installation a failed check`：侧栏和悬停提示；[`packages/app-shell/src/features/workspace/pending-update.test.tsx:215`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L215) `reports an installation whose package had to be discarded`、[`packages/app-shell/src/features/workspace/pending-update.test.tsx:253`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L253) `does not blame a later check from elsewhere on the installation`、[`packages/app-shell/src/features/workspace/pending-update.test.tsx:284`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L284) `names a failure by the operation the host reports, even with no checking in between`：失败后才挂载时仍是安装失败，`checking` 与检查失败合并到达时换成检查失败 | `Covered` |
| 失败时版本按钮可点、没有红点 | [`packages/app-shell/src/features/workspace/desktop-update-control.test.tsx:96`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/desktop-update-control.test.tsx#L96) `shows a failed check and leads to its reason without advertising a release`、[`packages/app-shell/src/features/workspace/desktop-update-control.test.tsx:115`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/desktop-update-control.test.tsx#L115) `does not call a failed installation a failed check`：按钮可点并调用跳转，没有红点。设置对话框切到更新日志页只在 [`packages/app-shell/src/features/settings/settings-dialog-release-notes.test.tsx:80`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/settings/settings-dialog-release-notes.test.tsx#L80) `opens the offered release above a history that does not list it` 里对 `ready` 和 `manual_update` 测过，没有 `failed` | `Partial` |
| 更新日志页显示原因并可重试 | [`packages/app-shell/src/features/workspace/pending-update.test.tsx:350`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L350) `explains a failed check and retries it`、[`packages/app-shell/src/features/workspace/pending-update.test.tsx:215`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L215) `reports an installation whose package had to be discarded`：显示原因，“重试”发起检查 | `Covered` |
| 安装器拒绝后可再次确认 | [`packages/app-shell/src/features/workspace/pending-update.test.tsx:189`](https://github.com/ora-space/desktop/blob/a7fb0ff1e1d16541ee6fd8c38c73e04db7b4ba36/packages/app-shell/src/features/workspace/pending-update.test.tsx#L189) `reports a rejected installation until it is confirmed again`：提示安装失败，再次确认后提示消失并再次安装。这条用例用普通 `Error` 拒绝，不是真实的 `internal_error` 契约错误；宿主在安装器拒绝后恢复 `Ready` 没有引擎用例 | `Partial` |
| 失败原因带原因链写进 Desktop 日志 | 尚无：没有断言日志内容的测试 | `Missing` |
| 真实应用里失败、重试 | 尚无，没有实测 | `Missing` |

### 决策依据

不来自本 ADR 的决策：这是 Desktop 更新入口的失败提示，和“可更新”区块共用设置里的同一入口，所以记在这里一起核对。ADR D3（更新入口与确认）。
