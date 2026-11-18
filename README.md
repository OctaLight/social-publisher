# 麦田社交发布系统（social-publisher）

基于 [CzzzzzzJ/mdldm-maitian](https://github.com/CzzzzzzJ/mdldm-maitian) 的本地化部署版本，适配 Windows 直连运行。

一个面向自媒体创作者的多平台视频内容自动分发工具，支持 B站、抖音、小红书、视频号、YouTube、X、Instagram 七大平台的一键填写与定时发布。

## 仓库说明

| 原始仓库 | 角色 | 关系 |
|---------|------|------|
| [mdldm-maitian](https://github.com/CzzzzzzJ/mdldm-maitian) | 中控 Hub（调度/防重/数据/Web大屏/飞书通知） | 主仓库，本版本在此基础上增加 Windows 支持和多项稳定性修复 |
| [mdldm-maitian-publisher](https://github.com/CzzzzzzJ/mdldm-maitian-publisher) | Chrome 插件源码（各平台填写引擎） | 作为依赖目录 `maitian-publisher/` 存在，被 hub 动态加载 |

**两个仓库不是重复，而是分层设计：** publisher 的 `lib/publish.js` 被 hub 动态加载，共享同一份适配器代码。

---

## 系统架构

```
config.json ──→ hub.mjs（入口，命令分发）
                ├─ driver.mjs    ← 驱动选择层（direct / plugin）
                │   ├─ chrome-shim.mjs  （direct 模式：Node 直连 CDP）
                │   └─ bridge.mjs       （plugin 模式：开发桥通信）
                ├─ db.mjs         ← SQLite 状态持久化（期数/任务/流量/日志/元数据）
                ├─ checks.mjs     ← 发布前查重（X/YouTube/Instagram/B站）
                ├─ feishu.mjs     ← 飞书卡片 + Webhook 通知（含去重）
                ├─ web.mjs        ← 管理网页（8787 端口）
                └─ publish.js     ← publisher 适配器（7 个平台填写引擎）
```

**两种驱动模式：**
- `driver: "direct"`（默认）— Node.js 直连专用 Chrome 调试端口（CDP），无需安装 Chrome 商店插件
- `driver: "plugin"` — 通过浏览器扩展面板的开发桥与已安装的 Chrome 插件通信

---

## 快速启动

```powershell
# 1. 确保环境
node -v   # 需要 ≥ 22.5

# 2. 首次安装向导（生成 config.json）
node src/init.mjs

# 3. 启动专用 Chrome（带调试端口）并打开各平台登录页
node src/hub.mjs login
# → 弹出专用 Chrome，里面打开了 B站/抖音/小红书/视频号/YouTube/X/Instagram 登录页
# → 在各页面完成扫码或账号密码登录

# 4. 体检确认各平台状态
node src/hub.mjs doctor

# 5. 启动管理网页
node src/web.mjs          # ← 直接运行即可（旧版只导出函数会静默退出，已修复）
# → 浏览器打开 http://localhost:8787/?key=<web.key>
```

### 配置要点（config.json）

```json
{
  "plugin": "local",
  "pluginDir": "<publisher 仓库绝对路径，如 /Users/xxx/mdldm-maitian-publisher>",
  "driver": "direct",
  "chromePort": 9333,
  "chromeProfile": "<Chrome 专用 profile 目录，如 ./chrome-profile>",
  "rootPath": "<你的视频工程文件夹>",
  "auto": ["youtube", "x", "instagram"],
  "manual": ["bilibili", "douyin", "xhs", "channels"]
}
```

| 字段 | 说明 |
|------|------|
| `pluginDir` | publisher 仓库的绝对路径 |
| `driver: "direct"` | 直连模式（不需要装 Chrome 插件） |
| `auto` / `manual` | 哪些平台全自动发布，哪些只填不发等你点 |

---

## 发布能力一览

| 平台 | 自动化程度 | 支持内容类型 | 说明 |
|------|-----------|-------------|------|
| X (Twitter) | ✅ 全自动 | 🎬 视频 / 🖼️ 图片 / ✍️ 纯文字 | 填文案 → 自动点 Schedule；支持长文+引用视频帖模式；images 字段替代 video 时走图片模式 |
| YouTube | ✅ 全自动 | 🎬 视频 | 查 Studio 列表核对 → 自动定时 |
| Instagram | ✅ 全自动 | 🎬 视频 / 🖼️ 图片（轮播最多10张） | 设定时 → 自动提交；图片模式下进入滤镜编辑页，无需 Reels 流程 |
| **B站** | ⚠️ 只填不发 | 🎬 视频 | 填好标题/封面/标签后停止，等你手动点「发布」；内置前置查重（按标题 API 查询投稿管理） |
| **抖音** | ⚠️ 只填不发 | 🎬 视频 / 🖼️ 图文（多图笔记） | 同上；图片模式会自动切到「图文」tab，上传多张图片后填入标题正文 |
| **小红书** | ⚠️ 只填不发 | 🎬 视频 / 🖼️ 图片（多图笔记） | 同上；images 替代 video 时走纯图文流程，跳过封面编辑步骤 |
| **视频号** | ⚠️ 只填不发 | 🎬 视频 | 同上（还会检测定时是否被平台篡改） |

> 国内四家"最后一下你点"是作者主动的风控设计，非功能缺失。

---

## 命令参考

```bash
node src/hub.mjs login               # 打开专用 Chrome + 各平台登录页
node src/hub.mjs doctor              # 体检：检查各平台登录态和上传页结构
node src/hub.mjs plan <目录>         # 登记分发计划（不上传）
node src/hub.mjs run <目录> [选项]   # 执行发布流程
                             --only a,b   只跑指定平台
                             --force      忽略已到位状态重新跑
node src/hub.mjs status [id] [--logs]        # 查看发布进度及操作日志
node src/hub.mjs sync <id>           # 取回已发布链接、更新状态
node src/hub.mjs metrics [id]        # 采集流量并发日报（飞书推送）
node src/hub.mjs followers           # 采粉丝数
node src/hub.mjs board               # 同步看板到飞书多维表格（Windows 跳过）
node src/hub.mjs review <id> [天数]  # 输出某期 N 天复盘（--notify 发飞书）
node src/hub.mjs weekly              # 周报（每周二自动发）
node src/hub.mjs daemon              # 守护进程：定时轮询 + 自动取链接 + 提醒
node src/hub.mjs submit <id> <平台>  # 手动提交某个平台（需专用 Chrome 里已填好）
node src/hub.mjs mark <id> <平台> <状态>   # 人工修改状态
node src/hub.mjs notify-test         # 测试飞书通知是否正常

node src/web.mjs                     # 独立启动管理网页（8787 端口）
node src/demo.mjs                    # 演示模式（假数据，看效果）
```

---

## 已修复问题记录（2026-11-18 累计）

### 🔴 P0 — 阻断性问题（已全部修复）

| 问题 | 影响 | 修复文件 | commit |
|------|------|---------|--------|
| `init.mjs` Chrome 检测硬编码 macOS 路径 | Windows 下 `node src/init.mjs` 报「没找到 Google Chrome」 | `src/init.mjs` | `26e6a26` |
| `hub.mjs` NAS 重挂用 macOS 专有命令 | Windows 下载不到 NAS 时脚本报错 | `src/hub.mjs` | `26e6a26` |
| `pruneLogs()` 90 天过滤形同虚设 | 总数 ≤ 5000 时所有旧日志全部保留，数据库无限增长 | `src/db.mjs` | `8c11e61` |
| `node src/web.mjs` 静默 exit 0 | README 写的启动命令实际不可用，Web UI 无法独立打开 | `src/web.mjs` | `8c11e61` |

### 🟡 P1/P2 — 中等优先级（已全部修复）

| 问题 | 影响 | 修复方式 | commit |
|------|------|---------|--------|
| `caffeinate` Windows ENOENT uncaughtException | X 长文模式下脚本异步崩溃 | 拆 platform 分支：macOS 用 caffeinate + `.on('error')`；Windows 直接 spawn node，同时挂 error 监听避免 async crash | `defaebe` |
| `busy()` 在 direct 模式恒 false | 无法阻止多期并发，可能同时触发两轮 Chrome 操作 | 实现 PID 锁文件方案：`run()` 写 `data/hub.pid`，`busy()` 检查另一进程是否存活；try/finally 保证异常退出时清锁 | `defaebe` |
| 飞书通知同一事件重复发送（after 发一次 + daemon 20min 提醒又发一次） | 同一完成事件收双通知 | hub.mjs 关键 notify 调用加显式 `_metaKey`（如 `after:${epId}:${platform}`、`remind:${epId}`），同一事件 across after()/daemon 精确互斥；通用路径仍靠 30min 内容窗口兜底，支持 quiet 验证码类跳过 | `defaebe` |
| `board.mjs lark-cli` 在 Windows 抛 ENOENT | 运行 `hub board` 直接崩溃 | 启动时检测 `process.platform === 'win32'`，不存在则 warn 并 return null | `defaebe` |
| publisher 上传等待循环中标签页丢失卡死 | Chrome 崩溃或被关标签后任务永远停在「上传中」 | `publish.js` 上传循环每轮先 `chrome.tabs.get(tabId)` 验证存活，丢失则标记异常并中断 | `6d7d78f`（publisher）|

### 🟢 增强项（部分已实现）

| 项目 | 状态 | 说明 |
|------|------|------|
| 每日自动 pruneLogs（daemon 每周日清理） | ✅ 已实现 | `pruneLogs(90, 5000)` 写入 daemon 循环，每次周日 9 点体检时触发 |
| B站前置查重 | ✅ 已实现 | `checks.mjs` 新增 `bilibiliExisting(title)` 调投稿管理 API，`hub run` 防重阶段调用 |
| X / YouTube / Instagram 查重 | ✅ 已有 | 原有的 `xExisting`、`ytRows`、`igExisting` 继续工作 |
| WIP：抖音/小红书/视频号查重 | ❌ 未实现 | 这三个平台的发布 API 限制较多，暂不支持自动查重 |
| E1 平台改版 DOM 告警 | ❌ 未实现 | 每次 preflight 额外检查关键元素指纹，连续失败发告警 |
| E2 Chrome profile 缓存清理 | ❌ 未实现 | 每月自动清理 cache/cookie |
| E3 ffprobe 视频格式校验 | ❌ 未实现 | 在 validateProject 阶段读取分辨率/码率/时长上限 |
| E4 Web UI 一键提交按钮 | ❌ 未实现 | 在 web.mjs 状态卡上加手动提交入口 |
| E5 多期并发队列 | 🟡 部分缓解 | direct 模式下 `hub.run()` 写 `data/hub.pid`，`busy()` 检查另一进程存活；若两个 run 几乎同时启动仍可穿透（非互斥队列）。单人本机可忽略；多人共用需改写成进程级互斥 |

---

## 测试验证

```powershell
node test_e2e.mjs    # E2E 集成测试（Chrome CDP ↔ publish.js 协同）
```

E2E 测试结果（实测）：
- ✅ Chrome 启动成功（CDP 端口 9333 连通）
- ✅ 插件 lib/publish.js 加载成功（7 个平台适配器：bilibili/xhs/douyin/channels/youtube/x/instagram）
- ✅ tabs.create/get/query/remove 全流程正常
- ✅ preflight 登录检测可正常读取页面 DOM 状态
- ✅ pruneLogs 日期过滤逻辑正确（4000 条旧日志测试场景）

### 图片内容发布（2026-11-18）

publish.json 新增可选字段 `images`（相对路径数组），替代 `video` 走图片发布流程：

```json
{ "version": 1, "id": "ep-xxx", "images": ["图片/a.jpg", "图片/b.png"], "platforms": { "x": {"text": "..."}, "xhs": {"title":"...","body":"..."} } }
```

| 平台 | 图片模式行为 |
|------|------------|
| X | 上传附件，等待缩略图出现，定时逻辑不变 |
| 小红书 | 批量上传图片进多图笔记，跳过封面编辑步骤 |
| Instagram | 进入滤镜编辑页，支持多图轮播（最多10张），不触发 Reels 流程 |
| 抖音 | 自动切到「图文」tab 再上传，封面设置跳过 |

---

## License

MIT（继承上游）
