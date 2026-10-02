# CatNative / webcat 安卓能力桥接 —— API 参考

> **CatNative**（WebCatX 自研原生桥接框架）：项目页面通过全局对象 `window.webcat` 调用安卓能力（预览与打包 APK 均支持，**无需引入**）。同步方法直接返回对象；异步方法返回 Promise，用 `await`。失败：同步返回 `{ok:false,error}`，异步 Reject（Error.message）。
> `window.webcat` 由原生在页面开始/完成时**异步注入**，可能晚于 `load`；建议在 `DOMContentLoaded`/`load` 之后调用，若仍取不到请**重试/轮询** `window.webcat` 再调用（详见 `webcat-quick.md` 的 `whenWebcatReady`）。

## webcat.fs —— 私有文件（应用私有目录，两端一致）
| 接口(中文名) | 类型 | 入参 | 出参 |
|---|---|---|---|
| `fs.readFile`(读取文件) | 异步 | path, encoding?('utf8'/'base64') | `{ok, content}` |
| `fs.writeFile`(写入文件) | 异步 | path, content, encoding? | `{ok, size}`（自动建父目录） |
| `fs.list`(列出目录) | 异步 | dir? | `{ok, items:[{name,isDir,size,modified}]}` |
| `fs.delete`(删除) | 异步 | path | `{ok}`（文件/空目录） |
| `fs.mkdir`(创建目录) | 同步 | dir | `{ok}` |
| `fs.exists`(判断存在) | 同步 | path | `{ok, exists, isDir}` |
| `fs.stat`(文件信息) | 同步 | path | `{ok, size, isDir, modified}` |
| `fs.readAsset`(读取项目文件) | 异步 | path（项目内相对） | `{ok, content}`（打包后只读） |
| `fs.openFile`(系统打开文件) | 同步 | path | `{ok}` |
| `fs.zip`(压缩) | 异步 | source, targetZip(.zip) | `{ok, size}` |
| `fs.unzip`(解压) | 异步 | zipPath, outDir | `{ok, count}` |
| `fs.move`(移动/重命名) | 异步 | source, target | `{ok}` |
| `fs.copy`(复制) | 异步 | source, target | `{ok}` |

## webcat.ext —— 公开文件（target=`"root"`全授权 / SAF handle）
| 接口(中文名) | 类型 | 入参 | 出参 |
|---|---|---|---|
| `ext.isFullAccess`(是否全授权) | 同步 | — | `{ok, granted}` |
| `ext.requestFullAccess`(开启全授权) | 同步 | — | `{ok}`（跳本应用设置） |
| `ext.pickDirectory`(选择目录) | 异步 | — | `{ok, handle}`（授权持久化） |
| `ext.revoke`(撤销授权) | 同步 | handle | `{ok}` |
| `ext.list`(列出) | 异步 | target, path? | `{ok, items}` |
| `ext.readFile`(读取) | 异步 | target, path, encoding? | `{ok, content}` |
| `ext.writeFile`(写入) | 异步 | target, path, content, encoding? | `{ok, size}` |
| `ext.delete`(删除) | 异步 | target, path | `{ok}` |
| `ext.mkdir`(建目录) | 同步 | target, path | `{ok}` |
| `ext.exists`(存在) | 同步 | target, path | `{ok, exists, isDir}` |
| `ext.stat`(信息) | 同步 | target, path | `{ok, size, isDir, modified}` |
| `ext.openFile`(系统打开) | 同步 | target, path | `{ok}` |
| `ext.zip`(压缩) | 异步 | target, source, targetZip | `{ok, size}` |
| `ext.unzip`(解压) | 异步 | target, zipPath, outDir | `{ok, count}` |
| `ext.move`(移动) | 异步 | target, source, dst | `{ok}` |
| `ext.copy`(复制) | 异步 | target, source, dst | `{ok}` |

> `target` 取 `"root"`（需“所有文件访问”，path 为绝对路径）或 SAF `handle`（path 相对授权目录）。

## webcat.media —— 媒体保存 / 选择
| 接口(中文名) | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `media.saveImage`(存相册) | 异步 | name, data, path? | `{ok, uri?, path?}`，默认存 Pictures |
| `media.saveToDownload`(存下载) | 异步 | name, data, path? | `{ok, uri?, path?}`，默认存 Download |
| `media.save`(自动) | 异步 | name, data, path? | 按扩展名自动图片→相册、其它→Download |
| `media.pickImage`(选图) | 异步 | format?, maxBytes? | `{ok,name,mime,size,base64\|uri\|path}` |
| `media.pickFile`(选文件) | 异步 | mime?, format?, maxBytes? | 同上，mime 默认 `*/*` |

- `data` 支持：裸 base64 / `data:image/png;base64,...` / http(s) URL（自动下载）。
- `path`：省略=默认集合；非 `/` 开头=该集合下子目录（如 `myapp/cards`）；`/` 开头=绝对路径直接写文件（需全盘权限）。
- `format` 默认 `base64`；`uri` 返回 content uri；`path` 尽力解析本地绝对路径，失败会提示改用 uri/base64。
- `maxBytes` 默认 15MB（仅 base64 上限，超限请改用 `format=uri`）。

## webcat.camera —— 相机
| 接口(中文名) | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `camera.takePhoto` | 异步 | save? | `{ok,name,size,base64}`；调系统相机（免 CAMERA 权限），save=true 同时存相册 |

## webcat.audio —— 音频播放
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `audio.play` | 异步 | `{url\|source,path,volume?,seek?}` | 播完 resolve `{ok,end:true}`；本地支持 fs/root/SAF，或 http(s) |
| `audio.stop` / `pause` / `resume` | 异步 | — | `{ok}` |
| `audio.setVolume` | 异步 | volume(0~1) | `{ok}` |
| `audio.seek` | 异步 | position(ms) | `{ok}` |
| `audio.isSupported` | 同步 | — | `{ok, supported}` |

## webcat.tts —— 文字转语音（无需权限）
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `tts.isSupported` | 同步 | — | `{ok, supported, speaking}` |
| `tts.speak` | 异步 | text, {lang?, rate?, pitch?, volume?, queue?} | 读完 resolve `{ok,end:true}`；`lang` 如 `"zh-CN"`；`rate/pitch` 默认 1；`volume` 0~1；`queue=true` 追加不打断 |
| `tts.stop` | 异步 | — | `{ok}` 停止并结束当前朗读 |
| `tts.pause` | 异步 | — | `{ok}` 暂停（系统 TTS 无原生暂停，等价停止） |
| `tts.resume` | 异步 | — | `{ok}` 从头继续最近一次内容 |

## webcat.record —— 录音（需打包勾选“录音/麦克风”）
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `record.start` | 异步 | `{source:'fs'/'root'/handle, path?, format?:'aac'\|'amr'}` | `{ok,started,path}`；缺省 fs `records/rec_时间戳.*` |
| `record.stop` | 异步 | — | `{ok,path,size,durationMs,format}` |
| `record.cancel` | 异步 | — | `{ok}` 丢弃 |
| `record.state` | 同步 | — | `{ok, recording}` |

## webcat.scan —— 扫码（需打包勾选“相机”）
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `scan.scan` | 异步 | — | `{ok,text,format}`（实时扫码；预览返回引导） |
| `scan.decodeImage` | 异步 | `{uri\|base64\|source,path}` | `{ok,text}` |
| `scan.generate` | 同步 | text, size? | `{ok,base64}` 生成二维码 PNG |

## webcat.clipboard —— 剪切板（同步）
| 接口(中文名) | 入参 | 出参 |
|---|---|---|
| `clipboard.get`(读取剪切板) | — | `{ok, text}` |
| `clipboard.set`(写入剪切板) | text | `{ok}` |

## webcat.share —— 分享
| 接口(中文名) | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `share.text`(分享文本) | 同步 | text, title? | 系统分享面板 |
| `share.file`(分享文件) | 同步 | source('fs'/'root'/handle), path, mime? | fs/root 走 FileProvider |

## webcat.app —— 应用
| 接口(中文名) | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `app.info`(应用信息) | 同步 | — | `{ok, packageName, versionName, versionCode, appName}` |
| `app.quit`(退出应用) | 同步 | — | `{ok}` |
| `app.openFile`(系统打开文件) | 同步 | source('fs'/'root'/handle), path | `{ok}` |
| `app.install`(安装) | 同步 | source, path | 需允许未知来源；SAF 先复制到 cache |
| `app.openAppSettings` | 同步 | — | 跳本应用设置 |
| `app.isInstalled` | 同步 | packageName | `{ok, installed, appName?, versionName?, limited}`（limited=未勾选“应用列表”，可能误报未安装） |
| `app.listApps` | 异步 | — | 需打包勾选 QUERY_ALL_PACKAGES；`{ok, apps:[{packageName,appName,versionName,versionCode,isSystem}]}` |
| `app.launchPackage` | 同步 | packageName | `{ok,launched}`；按包名打开（勾选“应用列表”更准） |
| `app.openUri` | 同步 | uri | `{ok,launched}`；任意 scheme/url |

## webcat.permission —— 权限
| 接口(中文名) | 类型 | 入参 | 出参 |
|---|---|---|---|
| `permission.check`(检查权限) | 同步 | perms: string[] | `{ok, granted[], denied[], undeclared[]}` |
| `permission.request`(申请权限) | 异步 | perms: string[] | `{ok, granted[], denied[], undeclared[]}` |

> `undeclared` = 未声明权限（打包时未勾选 / 不支持）→ 提示“打包时勾选”或“系统授予”。特殊权限 MANAGE 走 `ext.requestFullAccess`。

## webcat.sys —— 系统信息（同步）
| 接口(中文名) | 出参 |
|---|---|
| `sys.info` | `{ok, brand, model, androidVersion, sdkInt, screenWidth, screenHeight, density}` |
| `sys.network` | `{ok, online, type:wifi/mobile/none}` |
| `sys.battery` | `{ok, level, charging}` |

## webcat.screen —— 屏幕（同步）
| 接口(中文名) | 入参 | 出参 / 说明 |
|---|---|---|
| `screen.keepOn`(屏幕常亮) | on | `FLAG_KEEP_SCREEN_ON` 开/关 |
| `screen.setOrientation` | mode | auto/portrait/landscape/sensor |

## webcat.phone —— 电话短信（同步，免权限跳系统）
| 接口 | 入参 | 出参 |
|---|---|---|
| `phone.dial` | number | `{ok}` |
| `phone.sms` | number, text? | `{ok}` |

## webcat.qq —— QQ 跳转（同步）
| 接口 | 入参 | 出参 |
|---|---|---|
| `qq.openProfile` | qq: QQ号 | `{ok, launched}` |
| `qq.openGroup` | qq: 群号 | `{ok, launched}` |

## webcat.browser —— 系统浏览器（同步）
| 接口 | 入参 | 出参 |
|---|---|---|
| `browser.open` | url | `{ok}` |

## webcat.contacts —— 通讯录
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `contacts.list` | 异步 | — | `{ok, contacts:[{name, phones:[]}]}`（需 READ_CONTACTS，打包勾选） |

## webcat.location —— 定位
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `location.get` | 异步 | — | `{ok, latitude, longitude, accuracy, provider}`（需定位权限，打包勾选） |

## webcat.download —— 下载
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `download.download` | 异步 | url, source('fs'/'root'/handle), path | `{ok, path, size}` |
| `download.stream` | 异步 | `{url,source,path,headers?,onProgress}` | `{ok,path,size}`；带进度（中间帧 `{progress:{loaded,total}}`） |

## webcat.network —— 原生网络（无 CORS）
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `network.request` | 异步 | `{method,url,headers,body,timeout?,returnBase64?}` | `{ok,status,body}`；body 支持 `base64:` 前缀 |
| `network.upload` | 异步 | `{url,method?,headers?,fieldName?,files:[{source:'fs'/'root'/handle,path,name?}],fields?,onProgress}` | `{ok,status,body}`；multipart，已知路径原生直传 |

## webcat.page —— 页面窗口
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `page.open` | 同步 | `{url\|path, params?}` | `{ok}`；打开新页面窗口（path 相对当前网页根，或 http(s)/file 绝对） |
| `page.params` | 同步 | — | 本页收到的参数 |
| `page.end` | 同步 | — | 关闭当前页面 |

## webcat.store —— 会话共享（同步，仅内存，App 退出即清）
| 接口 | 入参 | 出参 |
|---|---|---|
| `store.set` | key, value | `{ok}` |
| `store.get` | key | `{ok, value}` |
| `store.remove` | key | `{ok}` |
| `store.keys` | — | `{ok, keys[]}` |

## webcat.statusBar —— 状态栏
> 默认：状态栏/底部导航栏由原生**自动避让**，普通页面**无需任何调用，也不要加 padding**（会重复留白）。
> `setAdaptive(true)` 沉浸时原生不再避让、状态栏覆盖页面；仅当顶部有需要完整显示的内容（标题/按钮等）才加 padding，且 `height` 是**设备px**、CSS 要 `height/density`。

| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `statusBar.setColor` | 同步 | color, darkIcon? | 自定义状态栏颜色（`#RRGGBB`/`#AARRGGBB` 或整数），导航栏同色；`darkIcon` 省略时按亮度自动判断 |
| `statusBar.setDarkIcon` | 同步 | darkIcon | `true`=深色图标，`false`=浅色图标 |
| `statusBar.hide` | 同步 | — | 隐藏状态栏（导航栏保留；焦点变化后保持隐藏） |
| `statusBar.show` | 同步 | — | 显示状态栏 |
| `statusBar.setAdaptive` | 同步 | enabled | `true`=沉浸（原生不再避让），`false`=恢复自动避让（默认）。别名 `setOverlay` |
| `statusBar.height` | 同步 | — | `{ok, height, density}`；`height` 为**设备px（非 dp）**，CSS 用 `height/density` |

```js
webcat.statusBar.setAdaptive(true);
const { height, density } = webcat.statusBar.height();
document.querySelector(".app-header").style.paddingTop = (height / density) + "px";
```
> `setAdaptive` 作用于整个预览/打包窗口（含原生顶栏）；导航栏颜色始终跟随状态栏。

## webcat.ui —— 提示（同步）
| 接口 | 入参 | 出参 / 说明 |
|---|---|---|
| `ui.toast` | msg, long? | `{ok}` |
| `ui.vibrate` | ms | `{ok}`；需打包勾选“震动” |

## webcat.flash —— 手电筒（同步，免权限）
| 接口 | 入参 | 出参 / 说明 |
|---|---|---|
| `flash.has` | — | `{ok, has}` 是否有可用闪光灯 |
| `flash.state` | — | `{ok, on}` 当前开/关 |
| `flash.on` / `flash.off` | — | `{ok, on}` 开/关 |
| `flash.toggle` | — | `{ok, on}` 切换 |

> 无需任何权限；相机被占用时返回 `camera_in_use`。页面关闭（release）时自动关灯。

## webcat.volume —— 音量（同步，免权限）
| 接口 | 入参 | 出参 / 说明 |
|---|---|---|
| `volume.get` | `stream?` | `{ok, stream, level, max, percent}` |
| `volume.set` | `level` 或 `{stream, level}` | `{ok, stream, level}` 设绝对档位 |
| `volume.up` / `volume.down` | `step?`（默认 1） | `{ok, level, percent}` 加/减 |
| `volume.getMax` | `stream?` | `{ok, max}` 最大档位 |
| `volume.isMuted` | — | `{ok, muted}` 媒体流是否静音 |
| `volume.setMuted` | `muted` | `{ok, muted}` 静音/取消 |

> `stream` 取 `music`(默认) / `ring` / `notification` / `alarm` / `voice` / `system` / `call`。

## webcat.brightness —— 亮度（同步，免权限）
| 接口 | 入参 | 出参 / 说明 |
|---|---|---|
| `brightness.get` | — | `{ok, value}` 当前窗口亮度 **0~100**，`-1`=跟随系统 |
| `brightness.set` | `value`(0~100；`<0`=跟随系统) | `{ok, value}` 设窗口亮度，立即生效 |

> 仅作用于当前预览/应用窗口，不修改系统亮度（无需权限）；Activity 重建后回默认。

## webcat.biometric —— 生物识别
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `biometric.isAvailable` | 同步 | — | `{ok, available, reason}`；`reason`=`none`/`no_hardware`/`hw_unavailable`/`none_enrolled`/`security_update_required`/`unsupported`/`unknown` |
| `biometric.authenticate` | 异步 | `{title?, subtitle?, description?, negativeText?, cancelable?, allowDeviceCredential?}` | 成功 resolve `{ok, authenticated:true}`；失败 Reject：`canceled`/`busy`/`no_hardware`/`not_enrolled`/`lockout`/`unavailable`/`error` |
| `biometric.cancel` | 同步 | — | `{ok}` 取消当前认证框 |

> 需打包勾选“生物识别”（`USE_BIOMETRIC`，普通权限，安装即授予、无运行时弹窗）。
> `allowDeviceCredential=true` 时允许 PIN/密码兜底（API30+ 用组合认证器，30 以下用 `setDeviceCredentialAllowed`），此时忽略 `negativeText`；PIN 兜底同样免权限。
> 认证框弹出期间再次调用 `authenticate` 返回 `busy`；页面关闭时自动取消。

```js
// 手电筒
webcat.flash.toggle();

// 音量（媒体流）
webcat.volume.up(1);
const { level, max } = webcat.volume.get();

// 亮度（0~100）
webcat.brightness.set(60);

// 生物识别（可加 PIN 兜底）
if (webcat.biometric.isAvailable().available) {
  try {
    await webcat.biometric.authenticate({ title: "验证身份", allowDeviceCredential: true });
    // 认证通过
  } catch (e) {
    // e.message = canceled / lockout / ...
  }
}
```

## webcat.image —— 图片处理（异步，免权限）
统一输入：`data`（裸 base64 / `data:image/...;base64,...` / http(s) URL）或 `source`+`path`（`fs`/`root`/SAF handle）。
统一输出：默认 `{ok, base64, width, height, size, format}`；给 `name` 则存相册，返回 `{ok, uri, path, width, height, size, format}`。

| 接口 | 入参 | 说明 |
|---|---|---|
| `image.info` | `{data?/source,path?}` | `{ok, width, height, size, format}` |
| `image.compress` | `{..., format:'jpeg'\|'png'\|'webp', quality:0-100, maxWidth?, maxHeight?, output?, name?, path?}` | 压缩（可同时限最大宽高） |
| `image.resize` | `{..., width?, height?, keepRatio?=true, ...}` | 缩放 |
| `image.crop` | `{..., x, y, width, height, ...}` | 裁剪 |
| `image.rotate` | `{..., degrees, ...}` | 旋转 |
| `image.flip` | `{..., horizontal?, vertical?, ...}` | 翻转（不传则默认垂直翻转） |

```js
const { base64 } = await webcat.image.compress({ data: canvas.toDataURL(), quality: 70, maxWidth: 1280 });
await webcat.image.crop({ data: base64, x: 0, y: 0, width: 200, height: 200, name: "avatar.png" });
```

## webcat.media —— 追加（多选 / 视频，免权限）
| 接口 | 类型 | 入参 | 出参 |
|---|---|---|---|
| `media.pickImages` | 异步 | `{format?, maxBytes?, limit?=9}` | `{ok, files:[{name,mime,size,base64?\|uri?\|path?}], truncated}` |
| `media.pickVideos` | 异步 | 同上（mime `video/*`） | 同上 |
| `media.saveVideo` | 异步 | `name, data, path?` | `{ok, uri?, path?}`（存 Movies） |
| `media.playVideo` | 同步 | `{url?/source, path?, title?, live?, orientation?='auto'\|'landscape'\|'portrait'\|'sensor'}` | `{ok}` 打开全屏播放页 |

> 多选走系统文件选择器（SAF，免权限）；`format` 可取 `base64`(默认)/`uri`/`path`，`maxBytes` 仅对 base64 生效。
> `media.playVideo` 播放页为平台播放器：支持本地/`file://`/`content://`/`http(s)` 渐进式（mp4 等），**不支持 HLS/DASH/RTSP/直播**；方向默认按视频有效宽高自动，可传 `orientation` 指定；页内带旋转/退出全屏按钮。

## webcat.sensor —— 传感器（同步/异步流，免权限，无计步）
| 接口 | 类型 | 入参 | 出参 / 说明 |
|---|---|---|---|
| `sensor.has` | 同步 | `type` | `{ok, has}` 是否有该传感器 |
| `sensor.get` | 异步 | `type` | `{ok, values, accuracy, ts}` 读取一次（约 1.5s 超时） |
| `sensor.watch` | 异步流 | `type, {rate?, limit?}, onProgress` | 中间帧 `{values, accuracy, ts}`；达到 `limit` 或 `unwatch` 时 resolve |
| `sensor.unwatch` | 同步 | `type` | `{ok}` 停止监听 |

- `type`：`accelerometer` / `gyroscope` / `magnetometer` / `light` / `proximity` / `pressure` / `orientation`
- `rate`：`normal`(默认) / `ui` / `game` / `fastest`；`values` 为数组（`orientation` 为旋转矢量）
- 是否真有该传感器取决于设备，先 `sensor.has` 判断

```js
if (webcat.sensor.has("accelerometer").has) {
  const p = webcat.sensor.watch("accelerometer", { rate: "ui", limit: 20 }, v => {
    console.log(v.values);
  });
  // await webcat.sensor.unwatch("accelerometer");  // 提前停止
}
```

## 安全限制
- 网页不可访问 WebCatX 自身目录（`/WebCatX/project`、`/WebCatX/cache`）：`ext.list` 命中返回空列表，其余读/写/删/开/分享命中返回“路径受保护，不允许网页访问 WebCatX 目录”；SAF 授权目录不受限。

## 示例
```js
// 私有文件
await webcat.fs.writeFile("data.json", JSON.stringify({a:1}));
const { content } = await webcat.fs.readFile("data.json");

// 压缩 / 解压（私有）
await webcat.fs.zip("js", "backup.zip");
await webcat.fs.unzip("backup.zip", "restore");

// 剪切板 / 系统信息（同步）
const { text } = webcat.clipboard.get();
const info = webcat.sys.info();

// 权限（异步）
const r = await webcat.permission.request(["android.permission.CAMERA"]);
if (r.undeclared.length) { webcat.ui.toast("该权限未在打包时勾选"); }

// 外部跳转
webcat.qq.openProfile("123456");
webcat.browser.open("https://example.com");

// 下载到私有目录
await webcat.download.download("https://x.com/a.png", "fs", "img/a.png");

// 通讯录 / 定位
const { contacts } = await webcat.contacts.list();
const loc = await webcat.location.get();

// 公开目录
if (webcat.ext.isFullAccess().granted) {
  await webcat.ext.writeFile("root", "/storage/emulated/0/Download/a.txt", "hi");
} else {
  webcat.ext.requestFullAccess();
}
const { handle } = await webcat.ext.pickDirectory();
await webcat.ext.writeFile(handle, "data.json", "{}");
```
