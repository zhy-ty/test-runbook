# 取证实现：chrome_devtools（CDP，连接真实 Chrome）

适用前提：本轮通道在环境文档#浏览器通道里被选为 `chrome_devtools`。它有三种角色，优先级不同：

1. **需求读取**：日常 Chrome 里的已登录页面（禅道等），读一次、落盘。
2. **诊断**：失败步的只读补充，≤5 次工具调用/≤30s；先看执行通道的 trace/screenshot，再开它。
3. **执行兜底**：只有没有本地 Playwright 工程时才用本通道执行；此时接受串行、用户不能同时操作、下载可能人工允许。纯人工目视、无法脚本化则实探。

**默认执行通道是本地 Playwright**；有 Playwright 就用 Playwright 执行，本通道只做需求读取/诊断。规则本体在 `references/templates/execution-contract.md` 的「取证规则」段，本文件只写这条通道的取法与坑。

## 执行兜底时的限制

- **串行**：一个 Chrome 默认 profile、一个标签页集合、一个下载目录；多份 runbook 并发会互抢标签页、登录态和下载文件。index 的并发分组只允许"只读 + 无共享数据键 + 独立 context/profile/下载目录"并行；本通道通常不满足，默认串行。
- **用户不能同时操作**：用户点一下就可能改变页面状态、抢焦点、让证据失效。需要用户同时用浏览器时，改用 Playwright 独立窗口。
- **宿主"允许自动化浏览器"**：宿主安全门，skill 没有权限关闭；每次新会话/新连接可能都要允许。记环境限制，让用户手动允许，不绕过、不把"点允许"写成测试步骤。若每次都弹且无法记住，优先改用 Playwright。
- **下载**：Chrome 可能弹"允许此网站下载多个文件？"，或把 zip 停在 `未确认 *.crdownload`。先在同机服务端路径直接校验产物；必须走浏览器下载时，按下面的下载节处理，每份产物最多 2 次。
- **企业策略**：`chrome://policy` 有 `DownloadRestrictions` 时，CDP 也可能被拦；记环境限制，不反复试。

## 执行档

本通道支持**合段断言**（inline `evaluate_script`），不要求也不产出可导出的脚本档；要沉淀跨需求复用的用例时，另接本地 Playwright 工程。CDP 的 inline 脚本只用于执行/诊断，不作为 `_lib` 资产。

- **合段断言**：单段 ≤5 步或 ≤30s；用轮询（`waitFor` 或短间隔重试）等条件成立，**不要串 setTimeout 嵌套**——超过默认超时会出现 `MCP error -32003/-32603`，那是通道现象，记台账类型"坑"，不要记成被测缺陷。
- 一次 `evaluate_script` 可以覆盖一段里多步的观测，但必须按 runbook 步骤号返回数组：`[{step, observed}, ...]`；任一步不符就停在那一步，后续标 ⏭。
- 断言只读渲染层。`document.querySelector` 读可见文本、`getComputedStyle` 读颜色、`document.title`/`location` 读 URL 都可以；**不要**用 `__INITIAL_STATE__`、Vuex/Pinia store、或直接 fetch 接口的返回值当界面观测。虚拟列表只渲染可见行，读数据源会判出用户根本看不到的"正确"。
- 失败即停，不靠重试把整段变绿；同一动作最多 2 次（含复现），连续 2 次同类失败即停。

## 诊断

- 只在失败步开；先看 Playwright/执行通道的 error、trace、screenshot；trace 解释不了才用 CDP 看 live DOM/console/network。
- 只读：`take_snapshot`、`take_screenshot`、`evaluate_script` 读 DOM、`list_console_messages`、`list_network_requests`；不要点击、填表、提交。
- ≤5 次工具调用/≤30s；结论写 close-out 的「诊断补充」，不改 runbook 的 ✅，不替代失败证据。
- **连对浏览器**：CDP 默认连的是用户日常 Chrome，不一定看得见 Playwright 启动的浏览器。要诊断 Playwright 的页面，让 Playwright 带 `--remote-debugging-port=9222` 启动，再让本通道/`connectOverCDP` 连过去；连不上就用 Playwright trace/screenshot，不反复试。
- 诊断不是换执行通道；修完必须回执行通道重跑失败步。

## 取什么

- 文案、URL、条数、状态值 → 取 CDP 快照（accessibility/DOM 文本），它才是判据；截图不能替代读到的文字。
- 视觉排版、颜色、图表形状 → `take_screenshot`，命名 `snap-R<n>-<步骤号>.png`。
- 下载、导出类 → 证据在文件系统里：记落地绝对路径与大小/行数/hash，页面提示只作辅证。
- 判定三步在写 runbook 时就拆进三列：**观测**（读哪个控件的哪段文本）→ **比对**（和「预期」哪一段逐字对）→ **落档**（写到哪个文件/快照）。
- **dev-smoke**：通过步 observed 写 `results.md` 一行；失败步才开 `snap-R<n>-<步骤号>.md`；视觉项才加 png。**full**：判定步与失败步全量，其余合并成段。

## 下载

- 先看同机服务端/生成目录是否已有 zip/docx/pdf；有就直接校验内容、大小、hash，不让浏览器下载。只有"下载行为"本身是验收项时才走浏览器。
- CDP 执行时，Chrome 可能把下载置为 `未确认 *.crdownload`（载荷完整但不落盘）。可在同一 Chrome 的 CDP 会话执行 `Browser.setDownloadBehavior = allow` 后重试一次；它按 tab 生效，同一 tab 的后续下载可能仍被拦，可换新 tab/新页面再试一次。
- 连续触发多个 zip 时，Chrome 可能弹"允许此网站下载多个文件？"；在源站设置里允许，或改用 Playwright。不要把"点允许"算进测试通过条件。
- 每份产物最多 2 次；超限就记 `blocked`/`观测缺失`，用服务端产物作辅证，不再反复导出。
- `Browser.setDownloadBehavior` 需要 MCP 暴露或本地 CDP 脚本；这是补丁，不是首选。若工具不提供，直接走服务端产物校验。

## 常见坑（这些形状一眼看不出没生效）

- **页面切换**：新开标签页/窗口后要先列标签页并切换，否则点到了、快照读的还是旧页面。
- **连接重建/page id 变更**：关闭 PDF 预览或弹窗后，Chrome 连接可能重建，page id 变化；重新 `list_pages`/`select_page` 再操作，页面状态通常还在。
- **iframe 内的表单**：快照读不到时先确认控件在不在 frame 里，并在该步「动作」列写清先切 frame。
- **对话框**：原生 alert/confirm 要单独处理；页面内自绘弹框按普通控件点。
- **虚拟列表**：只渲染可见行。要把某条数据当证据，先加"滚动/筛出让该行进入视口"的步骤，再取快照。
- **延迟渲染的页签/弹窗**：切页签后先等目标节点出现再读，不要用固定 sleep 猜。
- **按钮的条件显示**：`v-if`/`ifShow` 控制的按钮只在特定行或状态下出现；观测点要写明是在哪一行、哪个状态下看。
- **校验文案位置**：字段下方红字、toast、按钮禁用态是三种不同观测点，「观测点」列要指名是哪一种。
- **登录态掉线**：被踢回登录页时按环境文档#凭据条目重登，并往发现台账追加一行（类型=坑）。
- **截图落盘**：`take_screenshot`/`evaluate_script` 的 `filePath` 可能受 workspace roots 限制；先落系统临时目录，再用 Bash 复制到证据目录（这是工具链现象，记台账类型"坑"）。

## 目录与命名

```
docs/testing/YYYY-MM-DD-<topic>/runbooks/evidence/R<n>-<YYYY-MM-DD>/
  results.md                    # dev-smoke：通过步一行 observed
  snap-R<n>-<步骤号>.md         # 失败步证据（dev-smoke 只在失败时生成）
  snap-R<n>-<步骤号>.png        # 视觉类辅证
  data-backup/                  # 造数前的当前状态备份
```

同一 runbook 反复跑按日期分子目录，旧证据不覆盖——回归要靠前后对比判定修复是否生效。
