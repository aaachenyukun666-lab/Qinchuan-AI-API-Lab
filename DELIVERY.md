# AI Relay Meter — v0.1.0 交付报告

生成时间：2026-10-02 · 位置：`/root/AI-Relay-Meter/`

## 一、这是什么

依据《AI Relay Meter DeepSeek Harness 项目规格文档》，把原始的「AI Token
消耗工具.zip」（单文件 token 燃烧器）迭代为 **AI Relay Meter**：一个
OpenAI-compatible API 的 Token 计量与中转验真工具。中英双语、纯静态
HTML/CSS/JS + Node 工具链，可直接在浏览器跑，也可经 WebCatX 打包成 Android
APK。

## 二、逐文件交付（规格书 §23 要求）

| 文件 | 作用 | 验证 |
|---|---|---|
| `index.html` | 应用入口（单页） | 语法/结构审计通过，静态服务 200 |
| `src/js/core/usage.js` | usage 归一化（不伪造数字） | 单测通过 |
| `src/js/core/mask.js` | API Key 脱敏 | 单测通过 |
| `src/js/core/errors.js` | 17 类错误分类 | 单测通过 |
| `src/js/core/diff.js` | Token/费用分层差异计算 | 单测通过 |
| `src/js/core/stats.js` | 重复测试汇总/加权差异率/稳定性 | 单测通过 |
| `src/js/api.js` | OpenAI-compatible 客户端（含 SSE） | 集成测试通过 |
| `src/js/engine.js` | Token Usage Test 引擎（并发/中止） | 集成测试通过 |
| `src/js/verify.js` | 小额验真编排（余额对比/重复轮次） | 集成测试通过 |
| `src/js/report.js` | 报告构建 + JSON/CSV 导出 | 集成测试通过 |
| `src/js/i18n.js` | 双语运行时 + 离线 file:// 加载 | vm 冒烟通过 |
| `src/js/store.js` | localStorage 配置持久化 | vm 冒烟通过 |
| `src/js/app.js` | UI 控制器 | vm 全链冒烟通过 |
| `src/css/style.css` | Apple 风格样式 + 毛玻璃切片 | 静态审计通过 |
| `src/css/fonts.css` | 自托管 Inter 字体 | 加载验证通过 |
| `assets/fonts/*.woff2` | Inter 400/500/600 | 文件齐全 |
| `locales/{zh-CN,en-US}.{json,js}` | 双语包（各 153 键） | 键奇偶校验 0 差异 |
| `scripts/{serve,build-locales,live-check}.mjs` | 服务/构建/真机冒烟 | 语法+运行验证通过 |
| `tests/{core,integration}.test.js` | 37 个测试 | **37/37 通过** |
| `README.md` / `README.zh-CN.md` | 中英双 README | 子代理交付 |
| `LICENSE` / `CHANGELOG.md` / `LICENSE-THIRD-PARTY.md` | 许可与变更 | 子代理交付 |
| `docs/{ARCHITECTURE,DESIGN,webcat-*}.md` | 架构/设计/桥接文档 | 子代理交付 |
| `android/README.md` | WebCatX 打包步骤 | 子代理交付 |
| `demo/README.md` | 演示说明 | 本次新增 |

## 三、新增功能（规格书 §23）

1. OpenAI-compatible Base URL / Key / Model 配置 + 获取模型列表（/models 缺失不崩溃）。
2. 两种测试模式：Token Usage Test 与 Small-scale Verification（保守默认 3 次 / max_tokens 100 / 并发 1）。
3. usage 全字段记录：prompt / completion / total / cached / reasoning 及 details，缺失明确标记「API 未返回 Token Usage」。
4. 余额/额度前后对比：自动探测 + 手动输入兜底；Token 差异与费用差异分层。
5. 中性结论三态：未检测到明显差异 / 检测到差异建议重复测试 / 数据不足无法完整验真。
6. 重复验证（1/3/5/自定义轮次）+ 加权差异率 + 稳定性分级。
7. JSON + CSV 报告导出（密钥永不入报告）。
8. 完整 zh-CN / en-US i18n，切换不丢数据；file:// 离线可用。
9. Apple 设计语言：Action Blue #0066cc、17px 正文、600 字重标题、负字距、pill 按钮、单一投影、毛玻璃粘性导航（ablur 切片技术）、自托管 Inter。
10. 安全：密钥脱敏（`sk-abc********1234`）、导出前断言无密钥泄漏。

## 四、运行方式（规格书 §23「最短命令」）

```bash
cd AI-Relay-Meter
npm run build:locales   # 生成 locales/*.js（首次必做，file:// 离线依赖它）
npm run serve           # 打开 http://127.0.0.1:4173/
npm test                # 37 项测试
npm run test:live       # 真实 API 冒烟（需设置环境变量）
```

## 五、Android APK 构建（规格书 §23）

详见 `android/README.md`。要点：WebCatX 新建项目 → 放入本项目 `index.html`、
`src/`、`locales/`（含生成的 `.js`）、`assets/` → 启动页设 `index.html` →
打包权限按需勾选（本项目纯 fetch，无需原生桥权限）。

## 六、测试结果（规格书 §23「只报告实际执行过的」）

**实际执行并通过：**

| 测试 | 结果 |
|---|---|
| 语法检查（16 个 JS/MJS 文件） | 16/16 通过 |
| 单元测试（usage/diff/stats/errors/mask） | 23/23 通过 |
| 集成测试（mock 服务器端到端） | 14/14 通过 |
| 浏览器全链冒烟（vm 沙箱加载 12 模块） | 通过 |
| i18n 双语加载/切换/插值 | 通过 |
| 静态服务（HTML/JS/字体/语言包 200） | 通过 |
| 语言包键奇偶（zh == en，各 153 键） | 0 差异 |

**未执行（需你提供真实 Key）：** 规格书 §22 的「真实 API 请求、错误 Key、429、
DeepSeek 真实 usage」等网络验收。容器内无任何 API Key，我不会伪造结果。
提供 Key 后一条命令即可补跑：`ARM_BASE_URL=... ARM_API_KEY=sk-... ARM_MODEL=deepseek-chat npm run test:live`。

**已执行的错误场景覆盖（通过 mock 真实 HTTP 返回）：** 401（致命中断）、
404（models 不支持）、429/500/502（可重试分类）、无 usage、SSE 流、超时、CORS、
JSON 异常、余额缺失。

## 七、未完成问题（规格书 §23 要求诚实列出）

1. **真实 API 验收未跑** — 缺少 Key（唯一硬阻塞，见上）。
2. **Android 真机启动未跑** — 需 WebCatX 打包后在你的手机上验证；当前只验证了
   file:// 兼容性设计（经典 script + 生成的 locale 包）与静态服务。
3. **CSV 在 Excel 打开** — 已用 RFC 4180 + CRLF，但未在真实 Excel 里实测。

## 八、GitHub 发布建议（规格书 §24）

- Repository Name：`AI-Relay-Meter`
- Description：`AI API token meter and small-scale verification tool for OpenAI-compatible relay APIs.`
- 中文简介：`AI API Token 计量与 OpenAI-compatible 中转 API 小额验真工具。`
- Initial Release：`v0.1.0`
- Topics：`ai api token llm openai-compatible deepseek openrouter developer-tools observability api-testing`

## 九、Roadmap（规格书 §23）

- v0.1 MVP：已完成（本报告）。
- v0.2 Multi-provider：多服务商余额接口适配、价格表预设。
- v0.3 Advanced Verification：缓存/推理 token 计价、最小计费单位规则。
- v1.0 Stable：真实 API 全量验收 + Android 真机回归 + CI。

## 十、第三方素材（本轮新增）

- **Inter 字体**（rsms/inter，SIL OFL 1.1）→ `assets/fonts/`，SF Pro 开源替代。
- **ablur**（xujunhao940/ablur，MIT）→ 毛玻璃切片技术，已用纯 CSS 移植进粘性导航栏。
- **awesome-design-md**（VoltAgent，Apple 分析）→ `docs/DESIGN.md` 设计 token 来源。

三者均经 ghfast.top 镜像拉取，许可记录在 `LICENSE-THIRD-PARTY.md`。
