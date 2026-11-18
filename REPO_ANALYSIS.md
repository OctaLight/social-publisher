# xxai_feed_maitian 仓库完善性检查 & 多平台发帖优化建议

> 生成时间：2026-11-18 | 基于 `8ec33c4` commit

---

## 一、整体评估

| 维度 | 评分 | 说明 |
|------|------|------|
| 平台覆盖 | ✅ 优秀 | 7平台（B站/小红书/抖音/视频号/YouTube/X/Instagram）全覆盖，国内四家防翻车设计到位 |
| 防重机制 | ✅ 优秀 | X / YouTube / Instagram 均有完整查重；B站/抖音靠状态机 + 人工确认兜底 |
| 自动提交 | ⚠️ 中 | 仅 YouTube/X/Instagram 全自动；国内四家只填不发 |
| Windows 适配 | ❌ 待补 | 核心路径可运行，但安装向导和 NAS 挂载有 macOS 硬编码 |
| 测试体系 | ❌ 缺失 | 只有 1 个 md 解析测试，平台适配器无单测 |
| 容错恢复 | ⚠️ 弱 | 上传中途崩溃后需手动重启，没有断点续传 |
| 日志/监控 | ⚠️ 基本够用 | SQLite 落盘 + Web UI 查看，但没有日志轮转 |

---

## 二、Bug 及隐患清单

### 🔴 严重（会导致发布失败）

#### H1 — Bilibili.js 缺少关闭花括号
`maitian-publisher/lib/platforms/bilibili.js` 第 32 行附近 `schedule()` 函数体不完整，读出来的内容是：
```js
const got = await evaluate(T, `[...document.querySelectorAll('${TC} .date-show')].map(e=>e.textContent.trim()).join(' ')`);
// ← 这里应该还有 if/else + 函数闭合，但文件在 read 里只显示到这里
export function pluginDir() { ... }   // ← 这说明 export default 对象已经接上了
```
**影响：** 定时发布时 B 站的时间选择不生效（因为函数没正确闭合）。需要实际跑一次 `hub run` 验证。

#### H2 — init.mjs Chrome 检测硬编码 macOS 路径
```js
const chrome = ['/Applications/Google Chrome.app', path.join(os.homedir(), 'Applications/Google Chrome.app')].find(fs.existsSync);
chrome ? ok('Google Chrome') : bad('没找到 Google Chrome，请先安装');
```
**Windows 下永远找不到 Chrome**，会被误报为"没安装"。  
**修复方向：** 按 platform 分别检测：
```js
const chromePaths = process.platform === 'win32'
  ? ['C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe',
     path.join(process.env.LOCALAPPDATA || '', 'Google\\Chrome\\Application\\chrome.exe')]
  : ['/Applications/Google Chrome.app', path.join(os.homedir(), 'Applications/Google Chrome.app')];
```

#### H3 — hub.mjs NAS 重挂命令是 macOS 专属
```js
execFileSync('diskutil', ['unmount', alt], { stdio: 'pipe', timeout: 20000 });
execFileSync('osascript', ['-e', `with timeout of 20 seconds\nmount volume "${config.nasUrl}"\nend timeout`], ...);
```
Windows 上这两个命令不存在，守护进程启动时如果 NAS 掉线会直接 crash（虽然后面 catch 了，但会自动挂载逻辑完全失效）。  
**修复方向：** 加入 `process.platform === 'win32'` 分支，使用 `net use` 重连：
```js
if (process.platform === 'win32' && config.nasUrl) {
  const sharePath = config.rootPath.replace(/^[A-Z]:/, '\\\\' + nasHost + '\\');
  execFileSync('net', ['use', sharePath, '/persistent:no'], { stdio: 'pipe' });
}
```

---

### 🟡 中等（功能降级或体验问题）

#### M1 — 长视频上传中途崩溃无断点续传
`publish.js` 的 `run()` 函数在第 176 行起等视频上传完，但如果 Chrome 标签页被意外刷新、CDP 断开，`detach(st.tabId)` 之后整个批次需要从开头重来。  
**优化：** 在 `driver.mjs` 增加 `checkTabAlive(tabId)` 方法，上传前检查 tab 还在不在，不在就重建并跳过已完成的步骤。

#### M2 — B站/抖音/小红书前置查重缺失
X/YouTube/Instagram 在执行 `run()` 前都会先查平台上是否已有内容，B站/抖音/小红书没有。意味着如果误操作重新跑一次，三个平台都会再填一次表单（虽然不会真的重复发，但浪费时间）。  
**优化：** 在 `run()` 开头加入 `bilibiliExisting(title)` 调用投稿管理页接口核对。

#### M3 — 飞书通知存在重复发送场景
`after()` 函数在每种状态（awaiting_user / scheduled / published）都发飞书，而 daemon 每 20 分钟也会再次触发提醒。同一个事件可能收到两通知。  
**优化：** 用 `meta` key 去重（类似现有的 `learnTipDay` 模式），每个通知类型记录首次发送时间，20 分钟内不重复发。

#### M4 — 日志没有轮转，长期运行会撑爆 data/hub.db 的 logs 表
`db.mjs` 里 logs 只增不减，每期每条状态变化写一条，运行几个月后查询会变慢。  
**优化：** daemon 循环里加清理逻辑：保留最近 90 天的 log，或限制总行数（如最多 5000 条）。

#### M5 — lark-cli 依赖在 Windows 上无法工作
`board.mjs` 第 15 行调 `lark-cli` 命令行工具写入飞书多维表格，这是 macOS/Linux 工具。  
**影响范围：** 看板同步功能 Windows 不可用，其他功能正常。  
**缓解方案：** 启动 boardSync 前先检测 `lark-cli` 是否存在，不存在则跳过并记一条 warning。

---

### 🟢 增强项（锦上添花）

#### E1 — 平台 API 改版告警
目前适配器全靠 DOM 选择器工作，改版后静默失败（比如 X 改了 CSS class，`tweetTextarea_0` 找不到了），只有用户在网页上看到异常才知道。  
**建议：** 每次 preflight 时额外检查一个"关键元素存在性"指纹（如 `document.querySelector('.bcc-upload-wrapper')` 是否还在），如果连续 N 次体检找不到关键元素，发一条「XX 平台可能改版了」的告警。

#### E2 — 专用 Chrome profile 自动清理
`chrome-profile/` 目录会随时间膨胀（缓存、Cookie、IndexedDB）。目前没有自动清理策略。  
**建议：** 每月清理一次 `chrome-profile/Cache/`、`chrome-profile/Code Cache/` 和超过 30 天的 Cookie。

#### E3 — 视频格式自动校验
`scanProjects` 只检查 `.mp4/.mov/.m4v`，不校验分辨率、码率、时长上限。各平台对视频有不同限制（Instagram ≤ 20 分钟且 ≤ 1GB）。  
**建议：** 在 `validateProject` 阶段用 `ffprobe` 快速读取视频元数据，不符合平台限制时提前报错而不是跑到填表单那一步才失败。

#### E4 — Web UI 增加「一键重跑某平台」按钮
现在卡在某个平台时只能命令行 `hub submit <id> <platform>`，Web UI 里没有对应入口。  
**建议：** 在 `web.mjs` 的状态卡上加一个提交按钮（只对 X/YouTube/Instagram 显示）。

#### E5 — 多期并发控制
当前 `hub.run()` 发现插件里还有一轮在跑就直接 throw（`driver.mjs` busy 检查）。如果有两期同时计划发布，第二期的 run 会失败。  
**建议：** 支持队列化——第二期排队等第一期结束后自动续跑，而不是直接报错。

---

## 三、各平台适配器现状一览

| 平台 | fill | uploadDone | schedule | links | 前置查重 | 后置提交 |
|------|------|-----------|----------|-------|---------|---------|
| B站 | ✅ | ✅ | ✅ | ✅ API | ❌ | 手动 |
| 小红书 | ✅ | ✅(封面) | ✅ | ✅ DOM | ❌ | 手动 |
| 抖音 | ✅ | ✅ | ✅ | ✅ DOM | ❌ | 手动 |
| 视频号 | ✅ | ✅(封面) | ✅ | ✅ DOM | ❌ | 手动 |
| YouTube | ✅ | ✅ | ✅ | ✅ Studio | ✅ | ✅ 自动 |
| X | ✅ | ✅ | ✅ | ✅ DOM | ✅ | ✅ 自动 |
| Instagram | ✅ | ✅ | ✅ | ✅ Reels | ✅ | ✅ 自动 |

---

## 四、优先修复建议（按 ROI 排序）

| 优先级 | 问题 | 预计改动量 |
|--------|------|-----------|
| P0 | H2 — init.mjs Chrome 检测加 Windows 分支 | ~5 行 |
| P0 | H3 — hub.mjs NAS 挂载加 Windows 分支 | ~15 行 |
| P1 | H1 — 验证 B站 schedule 函数完整性，补全闭合 | ~10 行 |
| P2 | M1 — 上传中断检测（tab 存活检查） | ~30 行 |
| P2 | M2 — B站前置查重（调 member.bilibili.com 接口） | ~20 行 |
| P3 | M4 — logs 表定期清理（daemon 里加 pruning） | ~10 行 |
| P3 | E3 — validateProject 加 ffprobe 视频校验 | ~40 行（需 ffprobe 可用） |
| P4 | E4 — web UI 加「提交」按钮 | ~50 行 |

---

## 五、结论

**仓库主体质量很高**，平台适配器的 DOM 选择器经过大量实战打磨（注释里有精确的 bug 日期），防重逻辑也做得很细致。当前主要缺口集中在：

1. **Windows 可用性**（H2/H3）— 本次融合的 target 场景，最需要优先修
2. **容错恢复**（M1）— 长视频场景的痛点
3. **维护性**（M4）— 长期运行的累积问题

以上问题都不阻塞核心功能，可以逐步迭代修复。
