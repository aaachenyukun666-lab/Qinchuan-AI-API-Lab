# CatNative / window.webcat 速查

> **CatNative**：WebCatX 自研原生桥接框架。页面通过 `window.webcat.*` 调用安卓能力，预览与打包 APK 一致，**无需 import**。
> 同步方法直接返回对象（成功 `{ok:true,...}`，失败 `{ok:false,error}`）；异步方法返回 Promise（`await`），失败 Reject（Error.message）。
> 接口细节见 `docs/webcat-api.md`。

## 通用约定
- 返回信封：成功 `{ok:true,...}`；失败——同步 `{ok:false,error}`，异步 Reject。
- `window.webcat` 由原生在页面开始/完成时**异步注入**（可能晚于 `load`）。建议在 `DOMContentLoaded`/`load` 之后再调用；若仍取不到，请**重试/轮询** `window.webcat`（见下方 `whenWebcatReady`）。
- 权限三态：`granted` / `denied` / `undeclared`（未声明 = 打包时未勾选）。

## 默认行为与易错点
- 状态栏 / 底部导航栏由原生**自动避让**：普通页面**不要加 padding**；仅 `setAdaptive(true)` 沉浸且顶部需完整显示时才加，且 `height()` 是**设备px（非 dp）**，CSS 用 `height/density`。
- `media`：`path` 省略=默认集合；非 `/` 开头=该集合子目录；`/` 开头=绝对路径（需全盘权限）。`maxBytes` 仅对 base64 生效。
- `ext.target`：`"root"`（需全盘权限，path 绝对路径）或 SAF `handle`（path 相对授权目录）。
- 打包未声明权限的能力会返回引导性错误，需在打包权限清单勾选后再用。
- 网页不可访问 `/WebCatX` 目录（`ext.list` 返回空，读写等返回“路径受保护”）；SAF 授权目录不受限。

## 命名空间目录（细节见 webcat-api.md）
`fs` 私有文件 · `ext` 公开文件 · `media` 媒体保存/选择 · `camera` 相机 · `audio` 音频 · `tts` 文字转语音 · `record` 录音 · `scan` 扫码 · `clipboard` 剪切板 · `share` 分享 · `app` 应用 · `permission` 权限 · `sys` 系统信息 · `screen` 屏幕 · `phone` 电话短信 · `qq` QQ · `browser` 浏览器 · `contacts` 通讯录 · `location` 定位 · `download` 下载 · `network` 网络请求/上传 · `page` 页面窗口 · `store` 会话共享 · `statusBar` 状态栏 · `ui` 提示 · `flash` 手电筒 · `volume` 音量 · `brightness` 亮度 · `biometric` 生物识别 · `image` 图片处理 · `sensor` 传感器

## 桥就绪（异步注入）
`window.webcat` 是异步注入的，可能晚于 `load`。稳妥做法是等就绪后再调用：
```js
function whenWebcatReady(cb) {
  if (window.webcat) return cb(window.webcat);
  var n = 0, t = setInterval(function () {
    if (window.webcat) { clearInterval(t); cb(window.webcat); }
    else if (++n > 120) { clearInterval(t); console.warn("webcat 未注入"); }
  }, 50); // 最多约 6 秒
}
whenWebcatReady(function (webcat) {
  // 在这里调用 webcat.*
});
```

## 高频示例
```js
// 私有文件
await webcat.fs.writeFile("data.json", JSON.stringify({a:1}));
const { content } = await webcat.fs.readFile("data.json");

// 图片 dataURL 存相册子目录 / 选图转 base64
await webcat.media.saveImage("card.png", canvas.toDataURL(), "myapp/cards");
const { base64 } = await webcat.media.pickImage();

// 公开目录（全盘权限）
if (webcat.ext.isFullAccess().granted) {
  await webcat.ext.writeFile("root", "/storage/emulated/0/Download/a.txt", "hi");
} else {
  webcat.ext.requestFullAccess();
}

// 原生网络 / 流式下载
const r = await webcat.network.request({ method: "GET", url: "https://example.com" });
await webcat.download.stream({ url: "https://x.com/a.png", source: "fs", path: "img/a.png" });

// 权限
const p = await webcat.permission.request(["android.permission.CAMERA"]);
if (p.undeclared.length) webcat.ui.toast("该权限未在打包时勾选");

// 状态栏（默认自动避让，普通态不要加 padding）
webcat.statusBar.setAdaptive(true);
const { height, density } = webcat.statusBar.height();
document.querySelector(".app-header").style.paddingTop = (height / density) + "px";
```
