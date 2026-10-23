# test-runbook

仓库地址：**<https://github.com/zhy-ty/test-runbook>**

一个把**需求变成可交接的页面测试流程文档**的 skill。你一次给全前置环境和需求内容，它落一份环境文档、把需求切成验收条款、找到要测的页面和功能，再按"一条用户旅程一份文档"写出若干份 runbook。每份 runbook 都写成新会话照着做就能跑完的操作步骤——第一步打开冰箱门，第二步把大象放进去，第三步关上冰箱门，每一步都带"看哪里、期待看到什么、留什么证据"。

**默认 dev-smoke**：面向开发工程师的初步验证，不是专业测试。主流程冒烟、3–8 步/旅程（超过 10 步说明该拆旅程或切 full）、不做变体、不逐 step 取证；失败步才留可复核证据。要全量/完整回归时显式开 `full`。

执行通道默认**本地 Playwright 工程**（独立自动化 profile + auth.json），负责跑步骤、下载、取证；**CDP 只读诊断**只在失败步开，不替代判定；需求读取用日常 Chrome + CDP。这个分工是为了同时解决下载弹窗、并发隔离、用户同时用浏览器和 token 成本。

## 这个 skill 不承诺什么

- **不是测试框架，也不接 CI。** 执行者是 agent（你新开的那个会话），不是流水线。Playwright 是 agent 当场驱动的工具，不是 `npm test` 回放；判定步仍要逐条比对预期。
- **只覆盖 Web 页面操作。** 接口、SQL、后端日志、单元测试不在主步骤格式里。真要扩，`references/templates/runbook.md` 的列定义是唯一要动的地方。
- **不保证读得到你的需求系统。** 需求链接靠真实浏览器里的已登录状态打开抓取；读不到（未登录、无权限、页面结构不认识）就退回来请你贴正文，不会替你猜需求。
- **不绕过宿主/浏览器的安全确认。** "允许自动化浏览器""允许此网站下载多个文件"这类弹窗是安全门，skill 没有权限关闭；Playwright 通常不经过宿主的自动化授权，但仍要先验证，仍弹就记环境限制并手动允许。企业 `DownloadRestrictions` 同理。
- **不会自动开执行会话。** 宿主通常不能替用户来回开会话，所以去程靠你粘贴一句"先读契约，再按 `<runbook 路径>` 跑完"，回程靠你说一句"R1 跑完了，请读 close-out 收尾"。这句回程话没告诉用户，就等于没有回程。
- **生成会话不会替你跑。** 写完 runbook 就停手交给你，同一轮不得再开浏览器。同会话自跑只能在看到交付文案后、由你**显式选择**"自检轮"；它不构成交付验证。
- **0.3.0 不保证旧 runbook 兼容。** 现在还在优化测试阶段，合同字段会直接改；旧 runbook 请重新生成，不要补字段硬跑。

## 你需要准备的前置环境

| 项目 | 缺失的后果 |
| --- | --- |
| 被测入口：URL 或菜单路径 | **阻塞**。没有入口就没有第一步。 |
| 被测代码库的绝对路径（前端／后端，能给就给） | 决定失败诊断上限：有码 → 失败能查到文件与方法级；无码 → 只写到现象层并停下问你。有前端源码也是开合段断言/脚本档的前提。 |
| 测试账号与角色（说明用途即可，口令按来源记录） | 不阻塞本轮，但会卡在某一步的登录态上。 |
| 有效测试数据：什么数据算能测、怎么造或怎么找；要造数时的备份与处置方式 | 步骤里写不出"取法"，只能反过来问你，冷启动就破了。 |
| 本地 Playwright 工程（推荐执行通道：`playwright.config.*`、`channel: 'chrome'`、auth 方案） | 没有也能用 CDP 兜底执行，但会接受串行、用户不能同时操作、下载可能要人工允许。本地工程即使没接成 MCP 也算候选。 |
| 一条能读需求/能诊断的 CDP 通道 | 读不到需求系统就退回来请你贴正文；没有诊断通道时失败只能靠 Playwright trace/screenshot。 |
| 需求正文或需求链接 | 没有需求就没有验收条款，也无从判断漏测。 |

这些全部落进 `environment.md`。后续每个会话只引用路径和条目号，不抄正文；发现遗漏走"增补"，只改对应那一条。

## 工作流

```
步骤 0: 环境文档 + 测试档 + 浏览器预检 + 三角色通道（需求读取 / 执行 / 诊断）
步骤 1: 需求快照 → 验收条款清单（dev-smoke 只留验收相关摘录）
步骤 2: 定位被测对象 → 定位三态（代码确认／实探观察／用户口述）
步骤 2.5: 执行档决策（有 Playwright 默认合段断言/脚本档；无通道才实探）
步骤 3: 按用户旅程拆 runbook + 定轮次/并发分组/预算
步骤 4: 生成 execution-contract.md + 写 runbook（冷启动自检）
步骤 5: 交付后停手 → 你开新会话跑 → close-out 回来收尾
        多份默认串行；只读+无共享数据键+独立 context/profile/下载目录才并行
```

## 一轮怎么用

1. 本会话调用 `/test-runbook`，把前置环境和需求内容一次给它。得到 N 份 runbook、一份 `execution-contract.md` 和一段交付文案。
2. 默认**新开一个会话**，粘贴：

   ```text
   先读执行契约 <execution-contract 绝对路径>，再按 <runbook 绝对路径> 跑完这条旅程，环境文档 <绝对路径>
   ```

   只有你明确回复"在本会话自检轮"，生成会话才会继续跑；它会标自检轮，且不能当作冷启动验证。
3. 跑完或卡住后，回到第 1 步那个会话（编排会话）说：

   ```text
   R1 跑完了，请读 <close-out 绝对路径> 收尾
   ```

一份 runbook 对应一个会话。多份 runbook 默认串行；index 标了同一并行组、且满足"只读 + 无共享数据键 + 独立 context/profile/下载目录"时才并行。写操作或共享数据键（同一 id、同一 project、同一导出时间）一律串行。

## 产物

```
docs/testing/
  YYYY-MM-DD-<topic>/
    index.md                        # 续接索引：测试档、通道、并发分组、覆盖对照、旅程状态
    environment.md                  # 环境事实：入口、数据、代码、凭据、通道角色、预检、缺口
    requirement.md                  # dev-smoke 只留验收相关摘录；full 留全文
    execution-contract.md           # 本 topic 唯一的执行契约 + 取证规则
    discoveries.md                  # 只记会改变以后做法的发现
    runbooks/R1-<slug>.md           # 一份 = 一条用户旅程，头部引用 execution-contract.md
    runbooks/close-out/R1-<日期>.md # ≤1 页机器回程件
    runbooks/evidence/R1-<日期>/
      results.md                    # dev-smoke 通过步一行 observed
      snap-R1-<步骤号>.md           # 失败步证据
      snap-R1-<步骤号>.png          # 视觉项辅证
      data-backup/                  # 造数前的当前状态备份（有造数才建）
  _lib/<module>/<primitive>.md      # 原语卡（资产，跨需求复用）
tests/e2e/<module>/<primitive>.spec.ts  # 可执行用例（项目自己的 e2e 目录）
```

**脚本归属：** `_lib/` 的卡片和 `tests/e2e/` 的可执行用例都在**被测项目**里（或项目指定的独立 e2e 工程），随项目入库；本 skill 仓库只带格式与规则，不放任何项目脚本、路由、选择器或账号。

**两类文件的提交口径不同。** `YYYY-MM-DD-<topic>/` 是过程产物，默认不进提交清单；`_lib/` 是资产，应该入库。在被测项目里给忽略规则开口子时不能只写 `!docs/testing/_lib/`（父目录被排除时反选不生效），要写成：

```
docs/testing/*
!docs/testing/_lib/
```

## 关键机制（想改规则前先读这段）

- **两段式通道**：需求读取（日常 Chrome/CDP）｜执行（Playwright 独立 profile，默认）｜诊断（CDP 只读，失败步才开）。执行通道在一个 runbook 内定死；诊断不产出 ✅、不替代执行证据。
- **冷启动**是每份 runbook 的验收判据：一个全新会话只凭 index + 环境文档 + `execution-contract.md` + 这一份 runbook，不问人就能跑完并逐步判定。杀掉"换会话就跑不动"这个根因的门叫**死值或取法**——每个输入值要么写死，要么写清当场从哪儿取。
- **确定性**是第二条判据：通道、轮次、执行档、取证级别都在生成阶段定死并写进文档，执行期只读结论。同一份 runbook 两次跑执行通道不同，证据就不可比。
- **假通过**由三条规则挡住：只有当场观测到的事实与「预期」逐字/结构化一致才写 ✅；脚本只许读渲染层（可见 DOM／aria／toast／URL），读接口或前端 store 会看到用户看不见的东西；脚本只返回 `{step, observed}`，不宣布通过。
- **生成/执行分离**：写完 runbook 就停手，同一轮不得再开浏览器；同会话自检只能由你在交付后显式选择，并标"自检轮（不构成交付验证）"。
- **dev-smoke / full 分档**：dev-smoke 默认冒烟、3–8 步、不变体、通过步一行 observed、失败步才留证；full 才开变体、全量回归和判定步全量取证。两档都不生成用户报告。
- **并发**：默认串行；runbook 头部写"只读/写 + 共享数据键 + 账号 + 可并行组"，index 排班。只有只读 + 无共享数据键 + 独立 context/profile/下载目录才并行；写/共享数据一律串行。
- **预算与试错上限**：同一动作最多 2 次（含复现）；每份产物导出/下载最多 2 次；连续 2 次同类失败即停；诊断 ≤5 次/30s；dev-smoke 每份 runbook ≤20 次工具调用/15 min。超限记 `blocked`/`观测缺失`，不许一直试。
- **下载**：Playwright `waitForEvent('download')` + `saveAs` 优先；同机产物直接读服务端生成物；只有"下载行为"本身是验收项时才让浏览器完整走一遍。弹窗/`.crdownload`/企业策略记环境限制，不反复重试。
- **取证分级**：dev-smoke 通过步不截图、不 dump DOM，只写 `results.md` 一行；失败步才开 `snap-R<n>-<步骤号>.md`，视觉项才加 png。full 判定步与失败步全量、其余合并。
- **覆盖状态**只取 `✅实测／代码确认／观测缺失／本轮未开启／未覆盖／未跑`；代码确认和单样本不足不能写成 ✅，index 必须与 close-out/results.md 对得上。
- **需求外观察**：需求没写但看起来不对的现象，写 close-out 的「需求外观察」；你要求修复时只回归受影响步骤，未回归前相关条款不能标 ✅。
- **失败分叉**按代码可得性走两条路：有码查到文件:方法级、报你同意才改、改完只回归失败步与该条款的判定步；无码只写现象并标 `无码可查` 停下。
- **造数先备份**：动数据前先导出当前状态（以造数主键为根、关联表一起），结束按预先写好的处置方式回滚或删除，不把清理语句留给你当交付物；会生成/覆盖文件的动作，按环境文档「文件副作用」列备份或取得你的显式豁免。
- **通道探测含本地工程**：本地 Playwright 工程即使没有对应 MCP 也是**执行通道首选候选**；探测看 `playwright.config.*`、`npx playwright --version`、登录态/下载方案，不靠浏览器缓存目录判断装没装。
- **首跑校准**把文档缺陷和被测系统的 bug 分开：执行会话卡在"找不到控件、要的值没给、路径不存在"上，算 runbook 缺陷，改文档重发，不当 bug 记。
- **脚本原语先验后用**：复用前必须跑 probe（路由、控件、关键字面、下载目录），不过就标 `脚本失效`、回落实探并记台账；复用只限导航与造数，判定步必须本轮观测。
- **语义唯一源**：执行期义务与取证规则在 `references/templates/execution-contract.md`；编排语义在 `SKILL.md`；runbook 版式在 `references/templates/runbook.md`；产物版式在 `references/artifact-layout.md`；脚本卡格式在 `references/automation-scripts.md`；通道实现细则在 `references/browser-evidence*.md`。各管一段，别处不许再抄。

## 依赖

**没有外部 skill 依赖**，本仓库自带环境文档、执行契约与取证规则的定义。运行侧需要：

- **执行通道（推荐）**：本地 Playwright 工程，能驱动真实 Chrome（`channel: 'chrome'`），登录态用 storageState/auth.json，下载用 Playwright download 事件。
- **需求读取通道**：日常 Chrome + CDP，或你直接贴需求正文。
- **诊断通道（可选）**：CDP 只读；没有就用 Playwright trace/screenshot。

本 skill 是**手动调用**的（frontmatter 里 `disable-model-invocation: true`）：只有你打出 `/test-runbook` 才会触发，模型不会自己想起来用它，因此它也不占常驻上下文。副作用是：在被 @ 提及或 `/` 调用后，agent 若再尝试用 Skill 工具"激活"它，会收到 `cannot be activated via the Skill tool due to disable-model-invocation`——这是预期噪音，不是安装失败；SKILL.md 开头已写明不要再调。

## 方式一：CC Switch

CC Switch 按固定目录发现 skill，本仓库的 `skills/` 这一层就是给它用的。

| 字段 | 值 |
| --- | --- |
| 仓库地址 | <https://github.com/zhy-ty/test-runbook> |
| Owner | `zhy-ty` |
| Name | `test-runbook` |
| Branch | `master` |
| Subdirectory | `skills` |

添加仓库后安装 `test-runbook`，没有需要额外拉的上游仓库。

> 如果你把默认分支改成 `main`，记得同步这张表，以及你自己的 CC Switch 配置。

## 方式二：Claude Code Plugin

```bash
claude plugin marketplace add zhy-ty/test-runbook
claude plugin install test-runbook@test-runbook-marketplace
```

调用：

```
/test-runbook:test-runbook
```

CC Switch 装完后可能不带插件前缀，那 `/test-runbook` 就是同一个入口。

## 版本与更新

当前 **0.3.0**：

- 两段式通道：Playwright 默认执行，CDP 只读诊断；需求读取仍走日常 Chrome/CDP。
- 默认 dev-smoke：3–8 步、不变体、通过步一行 observed、失败步才留证；full 显式开启。
- 共享 `execution-contract.md`，runbook 不再内联契约，减少重复 token。
- 并发分组、预算与硬性试错上限、下载/弹窗处理、宿主"允许自动化"边界、无用户报告。
- close-out 压到 ≤1 页；`results.md` 承担通过步证据；基线只存摘要不存整份二进制。

**旧 runbook 不兼容**：0.3.0 改了 runbook 头部字段、取证列、契约位置和 close-out 版式。当前处于优化测试阶段，旧产物请重新生成，不要补字段硬跑。

**发布前检查：** committed 文档里引用的 `references/*.md` 必须都能在 `git ls-tree` 里找到。曾在早期版本漏提交新 reference 文件，导致换机/发布时悬空引用。

```bash
for f in $(grep -rhoE 'references/[A-Za-z0-9._/-]+\.md' README.md skills | sort -u); do
  git cat-file -e "HEAD:skills/test-runbook/$f" 2>/dev/null || echo "MISSING $f"
done
```

本仓库暂时没有 `docs/problem-log.md`。在它建起来之前，**改规则的动因写进 commit message**——加一条规则就说清它来自哪一次卡住，删一条就写下理由和日期，否则下一个人只会看到规则、看不出它为什么不能退回原来的写法。

## License

MIT，见 [LICENSE](./LICENSE)。
