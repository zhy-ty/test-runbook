# 取证实现：本地 Playwright 工程（默认执行通道）

适用前提：环境文档#浏览器通道里选中的是一个**本地 Playwright 工程**，并且已经验证它能驱动真实 Chrome。它不是 MCP 工具，但仍是合法的执行通道；步骤 0 必须记录工程绝对路径、版本、配置里的浏览器 channel、登录态方案、下载目录和并发能力。

## 角色与边界

- **执行 + 取证**：跑 runbook 步骤，返回 `{step, observed}`，保存失败现场。判定仍由执行会话逐条比对「预期」后写。
- **不是诊断通道的替代**：失败时先看 Playwright 的 error/trace/screenshot；需要 live DOM/console/network 时才开 CDP 诊断。诊断只读、≤5 次/30s，不改判定。
- **独立 profile**：执行用 Playwright 自己的临时/专用 profile + `auth.json`，**不要用用户日常 Chrome 的 `userDataDir`**。日常 Chrome 只用于读取需求或人工登录一次。这样用户可以在自动化期间使用自己的浏览器。
- **宿主"允许自动化浏览器"**：Playwright 走独立进程启动 Chrome，通常不触发宿主的自动化授权；但如果仍弹，记环境限制并让用户手动允许，不绕过、不写成测试步骤。这个弹窗不是脚本能修的。
- **不做 CI**：这是 agent 当场驱动的页面验证，不是 `npm test` 那种回放；脚本只负责观测，不宣布通过。

## 先探再判

在被测项目根或用户给的 e2e 工程目录执行：

```bash
npx playwright --version
npx playwright test --list
```

再读 `playwright.config.*`：

- `channel: 'chrome'`（或等价的真实 Chrome 配置）才算命中本机 Chrome；只装了 Playwright 包、没有可用浏览器时记 `未验证`。
- 登录态方案是 `storageState` 还是独立 profile；两种都要写进环境文档。
- 下载目录、`acceptDownloads` 是否配置；需要固定浏览器默认下载目录时用 `use.launchOptions.downloadsPath`，否则用 `download.saveAs()` 落到证据目录。
- `workers` 与并行度：默认 1；只有 index 标了并行组且满足"只读 + 无共享数据键 + 独立 context/profile/下载目录"时才调大。
- 浏览器缓存目录为空不代表没装：`channel: 'chrome'` 时 Playwright 本来就不下载 Chromium。

探测通过后，先跑一个最小 smoke，确认"能开页、能登录、能读到一个已知文本、能下载一个文件"，同时观察是否弹宿主"允许自动化浏览器"；弹就记环境限制，不绕过。再把结论写进环境文档。

## 推荐配置

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  // 单段 ≤5 步或 ≤30s；这里给整条 spec 一个上限，超时拆段
  timeout: 30_000,
  expect: { timeout: 5_000 },
  // 默认串行；并发按 index 的并行组显式开启
  fullyParallel: false,
  workers: 1,
  // 不用重试掩盖失败；失败就停在该步
  retries: 0,
  reporter: [
    ['list'],
    ['json', { outputFile: 'test-results/results.json' }],
  ],
  use: {
    baseURL: process.env.BASE_URL,
    // 用本机真实 Chrome，不下载 Playwright 自带 Chromium
    channel: 'chrome',
    headless: true,
    // 下载直接受理；通过 download.saveAs() 落到证据目录，不依赖浏览器下载栏
    acceptDownloads: true,
    // 失败留现场；视频默认关闭，避免证据膨胀
    screenshot: 'only-on-failure',
    trace: 'retain-on-failure',
    video: 'off',
  },
  projects: [
    { name: 'setup', testMatch: /auth\.setup\.ts/ },
    {
      name: 'chrome',
      use: {
        ...devices['Desktop Chrome'],
        channel: 'chrome',
        storageState: 'auth.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

## 登录态

优先用 setup project 自动登录：账号从 `E2E_USERNAME` / `E2E_PASSWORD` 环境变量读，登录成功后 `context.storageState({ path: 'auth.json' })`；业务 project 用 `storageState: 'auth.json'` 并声明 `dependencies: ['setup']`。

```ts
// tests/auth.setup.ts
import { test as setup, expect } from '@playwright/test';
import path from 'node:path';

const authFile = path.resolve('auth.json');

setup('登录并保存 storageState', async ({ page }) => {
  const baseURL = process.env.BASE_URL;
  const username = process.env.E2E_USERNAME;
  const password = process.env.E2E_PASSWORD;
  if (!baseURL || !username || !password) {
    throw new Error('缺少 BASE_URL / E2E_USERNAME / E2E_PASSWORD');
  }

  await page.goto(baseURL);
  await expect(page.getByPlaceholder('用户名')).toBeVisible({ timeout: 15_000 });
  await page.getByPlaceholder('用户名').fill(username);
  await page.getByPlaceholder('密码').fill(password);
  // Ant Design 会把“登录”渲染成“登 录”，用正则，不要精确文本
  await page.getByRole('button', { name: /登\s*录/ }).click();
  await page.waitForURL((url) => !url.href.includes('/login'), { timeout: 20_000 });
  await page.context().storageState({ path: authFile });
});
```

- `auth.json` 含 cookie/localStorage，加进 `.gitignore`，不要提交或粘贴到聊天里；登录态过期就重跑 setup。
- 只有验证码、SSO、动态口令这类无法脚本化的登录，才退回人工 `codegen` 或专用 profile 人工登录一次：

```bash
npx playwright codegen --save-storage=auth.json "<被测入口URL>"
```

- 用独立 profile 时不要和日常 Chrome 同时开同一个用户目录；会锁目录或丢登录态。执行用专用 `userDataDir`，与日常 profile 分开。
- 并发时**不要每个 worker 都跑 setup 登录**：setup 只跑一次，worker 复用同一份 `auth.json`。如果服务端是单会话策略，多 worker 共享同一 cookie 可以，但不要再登录一次；否则会互相踢下线。必要时改串行或给每个 worker 独立账号。

## 执行档

- **脚本档**：一段动作值得跨需求复用，且有稳定的原语卡/用例时用。用例放被测项目自己的 e2e 目录（如 `tests/e2e/<module>/<primitive>.spec.ts`）或项目指定的独立 e2e 工程，卡片放 `docs/testing/_lib/<module>/<primitive>.md`；**都不放 skill 仓库**。
- **合段断言**：一次性覆盖一段连续步骤时用临时 spec 或 `page.evaluate`，不沉淀卡片；跑完把 `{step, observed}` 写进证据。
- **实探**：没有 Playwright 执行通道、预期无法落渲染层断言，或登录态/通道不稳定时用。
- **单段 ≤5 步或 ≤30s**；超时拆段。任一步 observed 与预期不符就停在该步，后续标 ⏭，不要为了让整条 spec 变绿而加重试或 `soft`。

## 下载

Playwright 的下载事件是首选，不依赖浏览器下载栏：

```ts
// 一次点击 = 一个文件
const [download] = await Promise.all([
  page.waitForEvent('download'),
  page.getByRole('button', { name: /导\s*出/ }).click(),
]);
const target = path.join(evidenceDir, download.suggestedFilename());
await download.saveAs(target);
const failure = await download.failure();
if (failure) throw new Error(`下载失败：${failure}`);
```

- **先注册事件再点击**，否则可能漏掉下载。
- `download.saveAs()` 必须在浏览器关闭前完成；否则临时文件会被清掉。
- 一次点击触发多个文件时，用 `page.on('download', ...)` 收集，等到文件数或业务条件满足；不要用固定 `waitForTimeout` 猜。
- **同机产物优先直读**：导出文件在服务端/本机可读时，直接校验生成物（zip/docx/pdf 的内容、大小、hash）；只有"下载行为"本身是验收项时才让浏览器走一遍下载。runbook 里同时写下载路径和服务端兜底路径。
- 每份产物最多触发 2 次；失败先看 `download.failure()` 和目录，不要反复点导出。
- 企业策略可能限制下载：先看 `chrome://policy` 是否有 `DownloadRestrictions`；有就记环境限制，不绕过。
- 浏览器/宿主的"允许此网站下载多个文件"弹窗通常不会出现在 Playwright 的 download 事件里；如果仍出现，记环境限制，不要把它写成测试步骤。

## 并发与用户共存

- **隔离单位是 browser context**：每个 runbook 用自己的 context（或独立 Playwright 进程/独立 `userDataDir`）和独立下载目录；不要共用同一个 page 或同一个下载目录。
- **只读 + 无共享数据键**才可并行；写操作或共享数据键（同一 `id`、同一 project、同一导出时间）一律串行。index 的并发分组是权威，执行会话不得自己改。
- 并发时 `workers` 可以调大，但 setup 只跑一次；每个 worker 的 context 独立，`storageState` 共享。
- **用户共存**：Playwright 启的是自己的 Chrome 窗口（临时/专用 profile），用户可以用自己的日常 Chrome。headed 模式下用户看得到自动化窗口，但**不要操作它**；需要人工介入（验证码/SSO/目视）时按 runbook 的「人工介入点」暂停并交接。
- 连接日常 Chrome 的 CDP 模式不提供这种隔离：用户不能在自动化期间操作同一浏览器；需要共存就选 Playwright。

## 调试与诊断

- 失败先读 Playwright 的 error、`trace`、`screenshot`；`npx playwright show-trace <trace.zip>` 看调用链。
- 需要暂停观察时用 `PWDEBUG=1` 或 `page.pause()`；不要让暂停变成常态。
- 需要 live DOM/console/network 时，才开 CDP 诊断。可以让 Playwright 带调试端口启动：

```ts
// 仅诊断时开；端口不要和日常 Chrome 冲突
use: {
  launchOptions: { args: ['--remote-debugging-port=9222'] },
}
```

  然后让 chrome_devtools / `chromium.connectOverCDP('http://127.0.0.1:9222')` 连到**这个 Playwright 浏览器**，不要连日常 Chrome。连不上就用 trace/screenshot，不反复试。
- 诊断只读、≤5 次工具调用/≤30s、不改判定、不写 ✅；诊断结论写 close-out 的「诊断补充」。
- `trace`/`video` 可能含账号密码：不要提交、不要贴聊天；含密码的失败现场用完即删。

## 取证

- **dev-smoke**：通过步的 observed 写 `results.md` 一行；失败步写 `snap-R<n>-<步骤号>.md`；视觉项才加 png。通过步不截图、不 dump DOM。
- **full**：判定步与失败步全量，其余合并成段内证据；非判定步不得单独开文件。
- `test-results/` 是运行目录，不是交付证据；只把需要的失败现场/截图复制到 runbook 证据目录。
- 证据文件数 = 失败步数 + 视觉项数（dev-smoke）；超配就是取证过量，close-out 记一行。

## 硬约束

- 脚本只返回 `{step, observed}`，不返回通过/不通过；判定由执行会话逐条比对「预期」后写。
- `expect` 只能用于 probe/前置条件（页面能打开、控件存在），**不能用来宣布验收通过**。
- 只读渲染层：可见 DOM、aria、toast、URL、控件禁用态。禁止用 `page.request`、接口响应、localStorage/Vuex/Pinia store 代替界面观测。
- 单段 ≤5 步或 ≤30s；失败即停，不靠重试或 `soft` 变绿。
- 判定步必须本轮当场跑，不能引用历史 spec 的结果。
- 同一动作最多 2 次（含复现）；每份产物导出/下载最多 2 次；连续 2 次同类失败即停；诊断 ≤5 次/30s。

## 常见坑

- **headless/headed 差异**：视觉类判定用 headed 或明确记录 headless 的渲染差异。
- **新标签页/弹窗**：先 `context.waitForEvent('page')` 或 `context.pages()` 取到新页再操作，别继续用旧 page。
- **iframe**：先 `frameLocator` 定位，再读文本。
- **虚拟列表**：只渲染可见行，先滚动/筛选让目标行进入视口。
- **延迟渲染**：用 `waitFor`/`expect(locator).toBeVisible()` 等条件，不用固定 sleep 猜。
- **条件按钮**：`v-if`/`ifShow` 只在特定行出现；观测点要写明哪一行、什么状态下看。
- **中文按钮的可访问名**：Ant Design 等组件会在两个中文字之间插空格，例如按钮名是“登 录”而不是“登录”；用正则 `/登\s*录/`，不要用精确文本。
- **缓存与 keepAlive**：改库后页面可能不重新请求，先硬刷新或切路由再读。
- **下载事件时机**：先注册 `waitForEvent('download')` 再点击；`saveAs` 要在浏览器关闭前完成；多个下载用 `page.on('download')` 收集。
- **下载路径**：中文/空格文件名要加引号；同名文件会被覆盖，用时间戳或 runbook 指定目录。
- **企业下载策略**：`chrome://policy` 有 `DownloadRestrictions` 时，Playwright 也可能被拦；记环境限制，不反复试。
- **登录态过期**：重跑 setup；不要把新登录塞进每个 worker。
- **日常 profile 锁**：不要用日常 Chrome 的 `userDataDir`；会锁目录或丢登录态。

## 复用

用例/卡片必须带锚点（路由 + 组件 `文件:行` + 核对日期）和 3–5 条 probe。复用前先跑 probe；不通过标 `脚本失效`，回落实探，并记台账。同一原语只保留最新版，旧版删除并写“取代于 <日期>”。

卡片额外写清运行属性：

```markdown
## 运行属性
- 通道：Playwright <project 名称/路径>
- Profile：<独立 profile／auth.json>
- 下载目录：<...>
- 并发：<只读/写>｜共享数据键：<...>｜可并行组：<.../串行>
```

## 取数命令

```bash
npx playwright test <spec> --headed
npx playwright test --list
npx playwright show-trace test-results/<用例目录>/trace.zip
npx playwright show-report
```

证据落到 runbook 指定的 `runbooks/evidence/R<n>-<日期>/`；不要把报告目录当成交付证据，也不要把项目路径、账号、选择器写进 skill 仓库。
