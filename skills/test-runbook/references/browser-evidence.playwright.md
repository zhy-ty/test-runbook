# 取证实现：本地 Playwright 工程（不依赖 MCP）

适用前提：环境文档#浏览器通道里选中的是一个**本地 Playwright 工程**，并且已经验证它能驱动真实 Chrome。它不是 MCP 工具，但仍然是合法的执行通道；步骤 0 必须记录工程绝对路径、版本、配置里的浏览器 channel 和登录态来源。

## 先探再判

在被测项目根或用户给的 e2e 工程目录执行：

```bash
npx playwright --version
npx playwright test --list
```

再读 `playwright.config.*`：

- `channel: 'chrome'`（或等价的真实 Chrome 配置）才算命中用户日常浏览器；只装了 Playwright 包、没有可用浏览器时记 `未验证`。
- 登录态方案是 `storageState` 还是独立 profile；两种都要写进环境文档。
- `workers` 与并行度：多份 runbook/多个 worker 共用登录态会互抢会话，默认串行。
- 浏览器缓存目录为空不代表没装：`channel: 'chrome'` 时 Playwright 本来就不下载 Chromium。

探测通过后，先跑一个最小 smoke，确认“能开页、能登录、能读到一个已知文本”，再把结论写进环境文档。

## 执行档

- **脚本档**：一段动作值得跨需求复用，且有稳定的原语卡/用例时用。用例放被测项目自己的 e2e 目录（如 `tests/e2e/<module>/<primitive>.spec.ts`）或项目指定的独立 e2e 工程，卡片放 `docs/testing/_lib/<module>/<primitive>.md`；**都不放 skill 仓库**。
- **合段断言**：一次性覆盖一段连续步骤时用临时 spec 或 `page.evaluate`，不沉淀卡片；跑完把 `{step, observed}` 写进证据。
- **实探**：没有前端源码、预期落不到渲染层断言，或登录态/通道不稳定时用。

## 脚本的硬约束

- 脚本只返回 `{step, observed}`，不返回通过/不通过；判定由执行会话逐条比对「预期」后写。
- `expect` 只能用于 probe/前置条件（页面能打开、控件存在），**不能用来宣布验收通过**。
- 只读渲染层：可见 DOM、aria、toast、URL、控件禁用态。禁止用 `page.request`、接口响应、localStorage/Vuex/Pinia store 代替界面观测。
- 单段 ≤5 步或 ≤30s；超时拆段。任一步 observed 与预期不符就停在该步，后续标 ⏭，不要为了让整条 spec 变绿而加重试或 `soft`。
- 失败时保留 screenshot、trace、video；文字证据优先，截图只作辅证。
- 判定步必须本轮当场跑，不能引用历史 spec 的结果。

## 登录态

- 从真实 Chrome 导出 storageState，按“凭据按来源”记进环境文档；storageState 文件不进 git，过期就重新导出。

```bash
npx playwright codegen --save-storage=auth.json "<被测入口URL>"
```

- 在 `playwright.config.*` 里用 `storageState: 'auth.json'`；`auth.json` 加进 `.gitignore`。
- 用独立 profile 时不要和默认 profile 同时开同一个用户目录；会锁目录或丢登录态。
- 登录态失效按环境文档#凭据重登，并往发现台账追加一行（类型=坑）。

## 常见坑

- **headless/headed 差异**：视觉类判定用 headed 或明确记录 headless 的渲染差异。
- **新标签页**：先 `context.pages()` 取到新页再操作，别继续用旧 page。
- **iframe**：先 `frameLocator` 定位，再读文本。
- **虚拟列表**：只渲染可见行，先滚动/筛选让目标行进入视口。
- **延迟渲染**：用 `waitFor`/`expect(locator).toBeVisible()` 等条件，不用固定 sleep 猜。
- **条件按钮**：`v-if`/`ifShow` 只在特定行出现；观测点要写明哪一行、什么状态下看。
- **缓存与 keepAlive**：改库后页面可能不重新请求，先硬刷新或切路由再读。

## 复用

用例/卡片必须带锚点（路由 + 组件 `文件:行` + 核对日期）和 3–5 条 probe。复用前先跑 probe；不通过标 `脚本失效`，回落实探，并记台账。同一原语只保留最新版，旧版删除并写“取代于 <日期>”。

## 取数命令

```bash
npx playwright test <spec> --headed
npx playwright show-report
```

证据落到 runbook 指定的 `runbooks/evidence/R<n>-<日期>/`；不要把报告目录当成交付证据，也不要把项目路径、账号、选择器写进 skill 仓库。
