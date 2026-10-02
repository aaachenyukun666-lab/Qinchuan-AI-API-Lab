# AI Relay Meter

**AI API Token 计量与中转 API 小额验真工具**

**AI API Token Meter & Relay Verification Tool**

简体中文 · [English](README.md)

AI Relay Meter 是一款开发者工具：用于计量 OpenAI-compatible API 返回的 Token 用量，
并对中转 / 网关类接口执行小规模的使用量验真。

它会发送真实但刻意控制得很小的 API 请求，记录服务端**实际返回**的 `usage` 对象；
当服务商提供余额或额度数据时，再对比测试前后账户的变化。它产出的是可复核、可复现的
数据集，而不是一个结论。

---

## 目录

- [项目定位](#项目定位)
- [功能特性](#功能特性)
- [界面截图](#界面截图)
- [快速开始](#快速开始)
- [配置项](#配置项)
- [测试模式](#测试模式)
- [验真与差异](#验真与差异)
- [报告导出](#报告导出)
- [国际化](#国际化)
- [安全说明](#安全说明)
- [Android APK 打包（WebCatX）](#android-apk-打包webcatx)
- [开发](#开发)
- [测试](#测试)
- [Roadmap](#roadmap)
- [许可证](#许可证)

---

## 项目定位

大多数开发者访问大模型服务时，走的是中转或网关，而不是直接对接模型厂商。这一层很
方便，但它横在你和数字之间：响应里的 `usage` 对象与账户实际扣减的金额，是两个彼此
独立的观测值，没有任何机制保证它们一致。

AI Relay Meter 就是为这个差值准备的测量工具。它会：

1. 向你指定的 OpenAI-compatible `Base URL` 发送真实请求。
2. **按 API 原样**逐字段记录 Token 用量。
3. 可选地在测试前后各记录一次账户余额 / 额度读数。
4. 计算两者在 Token 层面与费用层面的差异。
5. 导出原始记录，任何人都能独立复核这些算式。

它是开发者工具、API 使用量测量工具和 API 测试工具。它不审计、不指控、不给服务商
排名。数据不完整时，它会如实说明，而不是靠猜测补全。

**适用对象：** 正在评估某个中转服务商的开发者、排查 Token 计费问题的集成方，以及
需要可复现用量数据（而非一张仪表盘截图）的任何人。

## 功能特性

- **以 OpenAI-compatible 为基准。** 任何实现 Chat Completions 协议的 `Base URL`
  都能用。不把任何单一服务商写死为唯一 Provider。
- **模型列表获取。** 通过 `GET /models` 生成可选列表。接口不存在或未授权时，自动
  降级为手动输入模型，而不是报错崩溃。
- **两种测试模式。** `Token Usage Test` 面向吞吐、延迟与请求稳定性；
  `Small-scale Verification` 面向低成本、保守参数的验真测试。
- **usage 全字段记录。** `prompt_tokens`、`completion_tokens`、`total_tokens`，
  以及服务商返回时的 `prompt_tokens_details`、`completion_tokens_details`、
  `cached_tokens`、`reasoning_tokens`。
- **不伪造数字。** API 未返回 `usage` 时，记录明确标记为
  `API 未返回 Token Usage` / `No Token Usage returned by the API`。估算值永远不会
  被当作实测值展示。
- **支持 Streaming 与非 Streaming。** 支持 SSE 流式请求，并在服务商返回时读取最终
  usage 帧。若流式响应结束时没有 usage，结果会被标注为可能不完整。
- **余额 / 额度对比。** 服务商提供前后余额接口时自动读取；没有时允许手动输入。
- **Token 与费用分开计量。** Token 差值与金额差值绝不混为一谈 —— 中转计费通常涉及
  倍率、输入输出分段定价、缓存价格、reasoning Token 与最小计费单位。
- **重复验证与汇总统计。** 支持 3 次、5 次或自定义次数的重复测试，输出平均 API
  Token、平均实际扣除、平均差异率以及最大 / 最小差异，便于判断差异是否稳定。
- **中性结论表达。** 结果只会表述为*未检测到明显计量差异*、*检测到 Token / 费用
  统计差异，建议进行重复测试*，或*当前数据不足，无法完成完整验真*。
- **JSON 与 CSV 导出。** 既有逐请求明细，也有会话汇总。
- **完整的中英双语国际化**，运行时可切换，且不会丢失配置、进行中的测试或测试结果。
- **密钥仅存本地。** API Key 保存在 `localStorage`，并在所有日志与报告中脱敏。
- **保守的默认参数。** 3 次请求、`max_tokens: 100`、并发 1。
- **随处可跑。** 既能作为纯静态页面在浏览器中运行，也能经 WebCatX 打包成 Android APK。

## 界面截图

> 截图占位。以下图片尚未从实际运行版本中截取 —— 首次发布 tag 之前，请用真实截图
> 替换每个占位文件。

| 界面 | 说明 | 文件 |
| --- | --- | --- |
| 首页 | API 配置、测试模式、参数、开始 / 停止 | `docs/images/main.png` *（占位）* |
| 实时结果 | 累计 Token、延迟、请求数、进度 | `docs/images/live-results.png` *（占位）* |
| 验真面板 | API 用量、账户用量、差异、差异率 | `docs/images/verification.png` *（占位）* |
| 报告页 | 会话汇总与逐请求明细、导出入口 | `docs/images/report.png` *（占位）* |

## 快速开始

### 环境要求

- **Node.js 18 或更高版本**（用于本地服务器、locale 构建与测试）。
- 现代浏览器；Android 打包则需要 WebCatX。
- 一个 OpenAI-compatible `Base URL` 与 API Key。

### 本地运行

```bash
git clone https://github.com/<your-org>/AI-Relay-Meter.git
cd AI-Relay-Meter
npm install
npm run build:locales
npm run serve
```

然后打开 <http://127.0.0.1:4173>。

`npm run build:locales` 会从 `locales/*.json` 生成 `locales/*.js`。克隆后请先执行
一次；否则 `file://` 场景下没有可加载的语言包。

### 三步完成首次测试

1. 填入 `Base URL`、`API Key`，也可以点 **获取模型列表** 直接选一个模型。
2. 选择 **Small-scale Verification**，保持默认参数不变
   （3 次请求、`max_tokens: 100`、并发 1）。
3. 点击 **开始测试**，然后打开 **查看详细报告** 并导出 JSON。

### 直接从磁盘打开页面

由于项目使用经典脚本（classic script）并预生成语言包，你也可以完全不开服务器，
直接用文件系统打开 `index.html`：

```bash
npm run build:locales   # 必须先执行一次，确保 locales/*.js 存在
xdg-open index.html     # 或用浏览器打开该文件
```

这与 Android 打包走的是同一条代码路径。原因见
[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)。

## 配置项

三个字段都位于首页的 API Configuration 卡片中。

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| **Base URL** | 是 | OpenAI-compatible API 的根地址，例如 `https://api.example.com/v1`。末尾斜杠会被归一化。程序会自动拼接 `/chat/completions` 与 `/models`。 |
| **API Key** | 是 | 以 `Authorization: Bearer <key>` 发送。仅保存在你设备浏览器的 `localStorage` 中。 |
| **Model** | 是 | 可手动输入，也可从 `GET /models` 的结果列表中选择。不写死任何模型。 |

**获取模型列表** 会发起 `GET {Base URL}/models`。若接口返回 401、403、404、空列表
或非法 JSON，程序会给出具体的失败原因，并始终保持手动输入模型可用。缺少 `/models`
接口绝不会阻塞测试。

配置会在本地持久化并在下次启动时恢复；清除站点数据即删除。语言选择单独存储，绝不
会重置你的 API 配置。

## 测试模式

### Token Usage Test

用于计量一般性的 Token 消耗、吞吐、延迟与请求稳定性。

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| 请求次数 | 3 | 本轮发起的请求数量 |
| Max Tokens | 100 | 每次请求携带的 `max_tokens` |
| 并发数 | 1 | 同时在途的请求数 |
| 模型 | — | 取自配置 |
| Prompt | 内置短 Prompt | 刻意写得很短，以限制消耗 |

每次请求记录：请求序号、时间、模型、Prompt Tokens、Completion Tokens、Total
Tokens、延迟（毫秒）、HTTP Status、Error。

### Small-scale Verification

保守模式，也是推荐的起步方式。它复用同一套请求引擎，只是默认参数刻意取小 ——
**3 次请求、`max_tokens` 100、并发 1** —— 并在其上叠加测试前后账户对比与差异分析。

该模式下的每一次请求都是**真实** API 调用。没有任何本地模拟，也不会凭空生成 Token
数字。

测试进行期间，**停止测试** 按钮始终可见。在途请求会被中止，已获得的部分结果会被
保留而不是丢弃。

参数允许用户自行调整。调高参数意味着真实的 API 消耗增加 —— 默认值的存在正是为了
避免误操作产生大量消耗。

## 验真与差异

验真运行时，界面会呈现以下量：

| 量 | 含义 |
| --- | --- |
| API-reported Token Usage / API 返回 Token | API `usage` 字段返回的 Token 数量 |
| Account Usage / 平台统计 Token | 服务商自有统计中的 Token 或额度数值（若可获得） |
| Difference / 差异 | `账户用量 − API 返回用量` |
| Discrepancy Rate / 差异率 | `\|账户用量 − API 返回用量\| / API 返回用量 × 100%` |

Token 层面与费用层面**分开报告**。Token 数量不是货币金额，工具绝不会把前者换算成
后者。当服务商存在倍率、输入输出分段定价、缓存价格、reasoning Token 定价、最小
计费单位或其他规则时，理论费用与实际费用会作为各自独立的字段记录。

结果只使用中性表达：

- **未检测到明显计量差异。** / No significant usage discrepancy detected.
- **检测到 Token / 费用统计差异，建议进行重复测试。** / A usage or billing
  discrepancy was detected. Repeated testing is recommended.
- **当前数据不足，无法完成完整验真。** / Insufficient data for full verification.

单次测试永远不会对服务商下结论。工具只做测量、记录、对比和展示。差异是否有意义、
是否在多次运行中保持稳定，交由你判断 —— 汇总数据旁边始终附带导出的原始数据，方便
独立复核算式。

若服务商没有提供余额或额度接口，你可以手动填入测试前后数值。如果必要数据完全缺失，
程序会明确提示无法完成完整验真，而不会给出估算结果。

## 报告导出

每次运行都会生成包含以下内容的报告：

- 测试时间、`Base URL`、模型
- 测试次数
- 总 Prompt Tokens、总 Completion Tokens、总 Tokens
- 若可获得：测试前额度、测试后额度、实际扣除、理论费用、实际费用、Token 差异、
  费用差异、差异率
- 数据完整性说明，列明哪些字段无法取得
- 除汇总数据外，同时提供逐请求明细

导出格式：

- **JSON** —— 完整的结构化记录，适合 diff 与归档。
- **CSV** —— 每个请求一行，另附汇总行，适合表格软件处理。

报告以当前语言渲染。API Key 绝不会以明文形式出现在报告中。

## 国际化

简体中文（`zh-CN`）与英文（`en-US`）在同一套代码中完整实现，而不是两个互相独立的
分支。

- 首次启动根据系统语言自动判断：`zh-CN` / `zh` 默认中文，其它语言默认英文。
- 语言切换入口位于首页右上角。
- 选择结果会在本地保存。
- 切换语言**不会**清空 API 配置、测试记录、Token 数据或测试结果，测试进行过程中
  也允许切换。测试数据不受影响。

所有用户可见的 UI、按钮、错误、帮助、设置与报告均已国际化。技术名称 —— API、
Token、OpenAI、DeepSeek、OpenRouter、JSON、HTTP、HTTPS、REST、SSE —— 按惯例
保留英文。

语言资源位于 `locales/`：

```text
locales/zh-CN.json   # canonical 源
locales/en-US.json
locales/zh-CN.js     # 由 `npm run build:locales` 生成，供 file:// 使用
locales/en-US.js
```

`zh-CN.json` 是 canonical 参考。新增文案时，先改它，再同步到 `en-US.json`，最后
重新生成。

## 安全说明

- **API Key 只在本地使用。** 密钥保存在你设备的浏览器 `localStorage` 中，并且只
  发送给你自己配置的 `Base URL`，绝不上传到其它任何地方。
- **不写死凭据。** 源码中不含任何 API Key 或服务商密钥。
- **日志脱敏。** 密钥在进入任何日志行、提示信息或导出报告之前都会被脱敏，例如
  `sk-1234************abcd`。仅保留很短的头部与尾部，足以辨认用的是哪把密钥。
- **报告可以安全分享。** 脱敏同样作用于报告输出，因此导出的 JSON 或 CSV 可以直接
  附在 issue 里。
- **无第三方埋点。** 程序不发起任何分析或追踪请求。对外流量只流向你填写的
  `Base URL`。
- **静态且可审计。** 除语言包生成外，没有打包器、没有构建期代码生成；`src/js/`
  里读到什么，运行的就是什么。
- **本地服务器仅监听回环地址。** `npm run serve` 绑定在 `127.0.0.1`。

API Key 关联着可计费账户。测试不熟悉的中转服务时，建议使用受限的或余额很低的密钥；
在共享机器上使用后，请清除站点数据。

## Android APK 打包（WebCatX）

本项目可通过 **WebCatX** 打包成 Android 应用。WebCatX 会把静态页面包进 WebView，
并通过 `window.webcat` 桥接暴露原生能力。无需 Android SDK、Gradle 或 Java 工具链。

### 打包步骤

1. **准备语言包。** 执行 `npm run build:locales`，确保 `locales/zh-CN.js` 与
   `locales/en-US.js` 已存在。缺少它们，打包后的应用无法离线加载翻译。
2. **组装打包目录。** 拷贝运行时文件 —— 清单见下。`node_modules/`、`tests/`、
   `scripts/` 不需要放进 APK。
3. **打开 WebCatX** 新建项目，将项目目录指向组装好的文件夹。
4. **设置启动页** 为 `index.html`（即 `appConfig.xlt` 中的 `启动页面` 字段）。应用
   是单页结构，不涉及路由。
5. **配置应用标识**：`appConfig.xlt` 中的 `appName`、`appPackageName`、
   `appVersionName`、`appVersionCode`。
6. **声明权限。** 验真流程只需要网络访问。仅当你扩展应用、需要把导出文件写入公开
   目录时，才添加存储权限；默认构建的导出文件保留在应用内部。
7. **在 WebCatX 中构建 APK**，然后安装到真机或模拟器。
8. **在设备上冒烟测试** —— 检查清单见下。

### 打包目录需要包含的文件

```text
index.html
assets/appIcon.png
locales/*.json
locales/*.js          # 生成产物，不要遗漏
src/css/style.css
src/js/**/*.js
appConfig.xlt
```

### `appConfig.xlt` 字段说明

随附的配置只是一个起点；下表说明各字段的作用。

| 字段 | 含义 | 建议值 |
| --- | --- | --- |
| `appName` | 启动器显示的应用名 | `AI Relay Meter` |
| `appPackageName` | Android 应用 ID；需全局唯一且跨版本稳定 | 你自己的反向域名 ID |
| `appVersionName` | 人类可读版本号 | `0.1.0` |
| `appVersionCode` | 单调递增整数；每次发版必须增大 | `10000` |
| `data.启用顶栏` | 是否显示原生标题栏 | `true` |
| `data.顶栏标题` | 标题栏文字 | `AI Relay Meter` |
| `data.顶栏颜色` | 标题栏颜色，格式 `#AARRGGBB` | 与应用主题一致 |
| `data.横屏` | 是否锁定横屏 | `false` |
| `data.显示状态栏` | 是否显示系统状态栏 | `true` |
| `data.启用进度条` | 是否显示原生加载进度条 | `true` |
| `data.进度条颜色` | 进度条颜色；`#00000000` 表示不可见 | `#00000000` |
| `data.支持打开外部应用` | 允许 `webcat.browser` / 深链跳出应用 | `true` |
| `data.提示打开外部应用` | 跳出应用前是否二次确认 | `false` |
| `data.启动页面` | 入口页面，相对项目根目录 | `index.html` |
| `indexPages` | 预加载进 WebView 的额外页面 | 留空 |
| `extraHeaders` | 注入页面请求的额外 HTTP 头 | 留空 |

### 本项目的注意事项

- **应用从 `file://` 加载。** 这正是 `src/js/` 采用 UMD 风格经典脚本、并且需要
  生成 `locales/*.js` 的原因：ES Module（`type="module"`）会被 WebView 在
  `file://` 下的 CORS 策略拦截，而经典 `<script src>` 可以离线正常加载。
- **网络访问由 WebView 发起。** 对已配置 `Base URL` 的请求必须能从设备访问到，并
  且必须允许跨域调用，因为页面来源是 `file://`。拒绝浏览器来源的服务商会在这里
  失败。WebCatX 另外提供 `webcat.network.request`，这是原生、无 CORS 的传输通道
  —— 见 [docs/webcat-api.md](docs/webcat-api.md) —— 也是需要对接这类服务商时的
  扩展点。
- **导出。** 从 `file://` 页面触发下载的行为与 HTTP 下不同。如果浏览器路径在设备上
  没有产生文件，可改走 `webcat.media.save` 或 `webcat.fs.writeFile`（两者均见
  [docs/webcat-quick.md](docs/webcat-quick.md)）。
- **状态栏间距。** WebCatX 已自动避让状态栏。除非你明确启用 `setAdaptive(true)`
  沉浸模式，否则不要给页头补 padding。

### 设备冒烟测试

安装 APK 后，确认以下各项：

1. 应用能启动并渲染首页（无白屏）。
2. 语言切换正常，且重启后仍然生效。
3. `获取模型列表` 成功；或失败时给出可读的提示。
4. 使用保守默认参数完成一次 `Small-scale Verification`。
5. **停止测试** 能中止在途运行。
6. 导出能产生可打开的文件。
7. 故意填错 API Key 会得到可读的 `401` 提示，且应用不崩溃。

完整的桥接参考见 [docs/webcat-api.md](docs/webcat-api.md) 与
[docs/webcat-quick.md](docs/webcat-quick.md)。

## 开发

### 环境要求

- Node.js 18+
- 现代浏览器

无打包器、无框架、无转译器。`src/js/` 由浏览器直接加载。

### 目录结构

```text
AI-Relay-Meter/
├── README.md              # 英文（GitHub 默认主 README）
├── README.zh-CN.md        # 中文
├── LICENSE                # MIT
├── CHANGELOG.md
├── package.json
├── index.html             # 单页应用入口
├── assets/appIcon.png
├── locales/               # zh-CN.json 为 canonical；*.js 为生成产物
├── src/
│   ├── css/style.css
│   └── js/
│       ├── app.js         # UI 装配与流程编排
│       ├── api.js         # OpenAI-compatible 客户端（含 SSE streaming）
│       ├── engine.js      # Token Usage Test 引擎
│       ├── verify.js      # Small-scale Verification 与差异分析
│       ├── report.js      # 报告构建 / JSON / CSV 导出
│       ├── i18n.js
│       ├── store.js       # localStorage 配置持久化
│       └── core/
│           ├── usage.js   # usage 归一化
│           ├── diff.js    # Token / 费用差异计算
│           ├── errors.js  # 错误分类
│           ├── mask.js    # API Key 脱敏
│           └── stats.js   # 重复测试汇总统计
├── docs/
│   ├── ARCHITECTURE.md
│   ├── webcat-api.md
│   └── webcat-quick.md
├── demo/                  # 本地 mock 服务器，供离线演示
├── android/               # WebCatX 打包配置与说明
├── scripts/
│   ├── serve.mjs          # 静态服务器
│   ├── build-locales.mjs  # locales/*.json -> locales/*.js
│   └── live-check.mjs     # 真实 API 冒烟检查
└── tests/                 # node --test 单元与集成测试
```

### npm 命令

| 命令 | 作用 |
| --- | --- |
| `npm run serve` | 在 <http://127.0.0.1:4173> 启动静态服务器 |
| `npm run build:locales` | 由 `locales/*.json` 生成 `locales/*.js` |
| `npm test` | `node --test tests/` |
| `npm run test:live` | 真实 API 冒烟检查 |

### 模块加载约定

`src/js/` 下的文件是 **UMD 风格的经典脚本**，不是 ES Module。每个文件都会把自己
挂载到 `window.ARM` 命名空间（`ARM.usage`、`ARM.api`、`ARM.engine` 等），同时保持
可被 Node 用 `require()` 加载以便测试：

```js
(function (root, factory) {
  if (typeof module === 'object' && module.exports) { module.exports = factory(); }
  else { root.ARM = root.ARM || {}; root.ARM.usage = factory(); }
})(typeof globalThis !== 'undefined' ? globalThis : this, function () {
  'use strict';
  // ...
  return { normalizeUsage: normalizeUsage };
});
```

因此 `package.json` **不设置** `"type": "module"`。这正是让同一份源码同时服务于
浏览器（通过 `<script src>`，在 `file://` 下离线可用）与 `node --test`（通过
`require`）而无需任何构建步骤的原因。完整理由见
[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)。

### 本地演示

`demo/` 提供了一个 mock 的 OpenAI-compatible 端点，用于离线开发。它让你能在不消耗
真实额度的情况下走通完整流程 —— 包括刻意缺失 `usage` 字段和注入错误状态码的场景。
把 `Base URL` 指向该 mock 服务器即可使用。

## 测试

项目提供两层测试。

### 单元与集成测试

```bash
npm test
```

对 `src/js/core/` 下的纯模块以及更高层模块运行 `node --test tests/` —— 覆盖 usage
归一化、差异计算、错误分类、脱敏与统计。由于模块是 UMD 形态，测试 `require()` 的
就是浏览器加载的同一份文件，不存在单独的测试构建产物。

### 真实 API 冒烟检查

```bash
export ARM_BASE_URL="https://api.example.com/v1"
export ARM_API_KEY="sk-..."
export ARM_MODEL="your-model-name"
npm run test:live
```

`scripts/live-check.mjs` 会向真实端点发起真实请求，并报告实际返回的内容。它
**不属于** `npm test`，也不在 CI 中运行，因为它需要凭据并会消耗真实额度。请使用
小额密钥与保守参数。

### 手动验证清单

以下行为需要真实端点或真机，需人工验证：

- 对 OpenAI-compatible 端点发起真实请求，含 `usage` 解析。
- 自定义 `Base URL`。
- 错误的 API Key（`401`）。
- HTTP `429` 处理。
- 不返回 `usage` 的端点。
- 使用保守默认参数运行 `Small-scale Verification`。
- 连续多次重复测试及其汇总。
- 中文完整流程。
- English 完整流程。
- 运行时 `zh-CN` ↔ `en-US` 切换，确认数据不丢失。
- Android 启动与核心流程。

> 本节只说明如何运行测试，刻意不声称测试数量或通过数量；当前结果请以实际执行上述
> 命令的输出为准。

## Roadmap

### v0.1 MVP

- OpenAI-compatible `Base URL` / API Key / Model 配置
- 模型列表获取与优雅降级
- `Token Usage Test` 与 `Small-scale Verification` 两种模式
- usage 全字段记录，并对无法获取的字段明确标注“未知”
- 测试前后余额 / 额度对比与差异率
- 重复验证与汇总统计
- 中性结论表达
- JSON 与 CSV 报告导出
- 完整的 `zh-CN` / `en-US` 国际化
- API Key 本地存储与日志脱敏
- 保守默认参数（3 次请求、`max_tokens` 100、并发 1）
- WebCatX Android 打包

### v0.2 Multi-provider

- 具名 Provider 配置档案，可保存与切换
- 服务商能力探测（端点实际返回哪些可选字段）
- Chat Completions 之外的更多请求形态
- 相同参数下对两个端点做并排对比

### v0.3 Advanced Verification

- 定时与间隔化的重复验证
- 差异率跨多次运行的趋势图
- 定价规则建模：输入输出分段、缓存价格、reasoning Token、倍率、最小计费单位
- 更大样本量下的统计显著性指标
- 多 Prompt 对比，以发现与 Prompt 相关的行为差异

### v1.0 Stable

- 冻结数据格式，并为导出报告提供迁移方案
- 为报告 schema 提供稳定性保证
- 扩展纯模块的自动化测试覆盖
- 无障碍与响应式布局梳理
- 发布正式版 APK

Roadmap 只是意向，不构成承诺，可能调整。

## 许可证

MIT —— 见 [LICENSE](LICENSE)。

Copyright (c) 2025 AI Relay Meter Contributors.
