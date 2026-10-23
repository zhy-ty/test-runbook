# 取证实现（生成侧）

取证**规则**（级别、命名、判定口径）只在 `references/templates/execution-contract.md` 的「取证规则」段；执行会话读的是实例化到 topic 目录的那一份。本文件只回答三件事：本轮用哪条通道、通道之间怎么分工、该通道下怎么把证据取到。

## 三种角色

一个 topic 可以同时用到三种角色，但每个 runbook 的**执行通道只有一个**：

- **需求读取**：日常 Chrome + CDP（或用户贴正文）。只读、一次性，读到的需求落盘后不再依赖登录态。
- **执行**：负责跑步骤、产出判定证据。默认本地 Playwright 工程（独立自动化 profile + storageState/auth.json）；没有 Playwright 时才用 CDP。
- **诊断**：只在失败步、且执行通道的 error/trace/screenshot 解释不了时开。CDP 只读诊断，不产出判定、不改 runbook 的 ✅，诊断结论写 close-out 的「诊断补充」。

## 选通道

先读环境文档#浏览器通道，按角色选，**不要按"谁已挂载"排序**：

- **执行通道默认 Playwright**：本地工程已验证（`playwright.config.*`、`npx playwright --version`、一次 smoke、`channel: 'chrome'`、storageState/独立 profile）就用它执行；下载/导出、多 runbook 并发、用户要同时用浏览器、长流程是它更适合的理由，不是前置条件。它的 download 事件、隔离 context、独立 profile 直接覆盖这些场景。
- **CDP 执行只在**：没有本地 Playwright，才用 CDP 兜底；纯人工目视、无法脚本化则实探。CDP 执行要接受串行、用户不能同时操作、下载可能人工允许。
- **需求读取**用日常 Chrome + CDP 或用户正文；不要为了读需求把执行 profile 切回日常 profile。
- **诊断通道**：优先 Playwright trace/screenshot；需要 live DOM/console/network 时才用 CDP。诊断 ≤5 次工具调用/≤30s，只读，失败步才开。

通道的可用性结论只在步骤 0 探测一次并固化。执行期发现执行通道不可用，停下报用户，不临场换执行通道——换了通道，证据格式和等待行为都会变，同一份 runbook 的前后两轮就不可比。诊断不是换通道，它不改变判定。

## 浏览器弹窗与下载

这些是浏览器/宿主行为，不是被测缺陷，也不是 runbook 的测试步骤：

- **宿主"允许自动化浏览器"**：宿主安全门，skill 没有权限关闭。Playwright 走独立进程启动 Chrome，通常不经过这条授权；如果仍弹，记环境限制并让用户手动允许，不绕过、不把它写成步骤。CDP 执行时每次新会话可能都要允许，这是选 CDP 的代价。
- **"允许此网站下载多个文件？"**：Chrome 对连续自动下载的保护。Playwright 的 download 事件通常不弹；CDP 执行时在源站设置里允许，或用 `Browser.setDownloadBehavior` 兜底。不要把"点允许"算进测试通过条件。
- **`未确认 *.crdownload`**：Chrome 已取回但等待确认。CDP 可在同一页面设 `Browser.setDownloadBehavior=allow` 后重试一次；Playwright 用 `download.saveAs()` 直接落盘。
- **企业下载策略**：先看 `chrome://policy` 是否有 `DownloadRestrictions`。有就记环境限制，不反复试；skill 不绕过企业策略。
- **同机产物**：导出文件在服务端/本机可读时，直接读生成物（zip/docx/pdf）校验内容；只有"下载行为"本身是验收项时，才让浏览器完整走一遍下载。每份产物最多触发 2 次。

## 通用要求

- 一条证据要能独立回答"这一步看到的值是什么、和预期差在哪"，不依赖对话上下文。`results.md` 的 observed 也必须是具体值（文案、URL、条数、状态、表头数组），不能写"正常"。
- 先等再判：判"没出现"之前，按该步的动作等一次并记等待时长，再重试同一观测；两次都无才判 ✗。偶发成功走契约 R4a。
- 读文字优先于截图；截图只作辅证，除非该步的预期本身就是视觉属性。
- 虚拟列表、骨架屏、延迟渲染的页签都会制造"元素不存在"的假失败——这三类观测点写 runbook 时就要带上"等什么出现"的前置条件。
- 取证按测试档分级：**dev-smoke** 通过步只写 `results.md` 一行，失败步才开 `snap-R<n>-<步骤号>.md`，视觉项才加 png；**full** 判定步与失败步全量、其余合并成段，非判定步不得单独开文件。
- **Token**：一段连续步骤尽量一次脚本调用跑完，只回 `{step, observed}`；通过步不截图、不 dump DOM；失败时先读 error/trace，再决定是否开 CDP 诊断。

## 按通道的实现

- 本地 Playwright 工程：见 `references/browser-evidence.playwright.md`。
- `chrome_devtools`（CDP）：见 `references/browser-evidence.chrome-devtools.md`。
- 其他具备语义定位 + 自动等待 + 脚本导出的通道：接入时新增一份同构的实现文件，只写该通道的取法与坑，不改上面的规则。没有实测过的通道不许先写"应该可以"。
