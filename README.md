# AI Media Studio — 自媒体 AI 内容创作与发布系统

AI Media Studio 是一款面向自媒体创作者的 **AI 内容创作与发布系统**。

本项目基于 [CzzzzzzJ/mdldm-maitian](https://github.com/CzzzzzzJ/mdldm-maitian) 进行深度二次开发，对代码结构、功能模块及运行方式进行了全面重构，形成完全独立的本地化部署版本。

---

## 主要特点

- **AI 辅助内容创作** — 支持 AI 撰写多平台文案、自动填充视频元信息
- **多平台一键发布** — B站、抖音、小红书、视频号、YouTube、X、Instagram，7 大平台统一管理
- **本地化部署** — 全链路私有部署，无需依赖任何云端环境
- **Windows 原生支持** — 针对 Windows 系统全面适配，开箱即用
- **持续迭代优化** — 根据实际使用需求不断修复问题、完善功能

> 📌 **说明：** 本项目是在原开源项目基础上进行深度二次开发的独立版本，具体功能及实现以当前仓库代码为准。

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
node src/web.mjs          # ← 直接运行即可
# → 浏览器打开 http://localhost:8787/?key=<web.key>
```

### 配置要点（config.json）

```json
{
  "plugin": "local",
  "pluginDir": "<publisher 仓库绝对路径>",
  "driver": "direct",
  "chromePort": 9333,
  "chromeProfile": "./chrome-profile",
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
| X (Twitter) | ✅ 全自动 | 🎬 视频 / 🖼️ 图片 / ✍️ 纯文字 | 填文案 → 自动定时；支持长文+引用视频帖模式 |
| YouTube | ✅ 全自动 | 🎬 视频 | 查 Studio 列表核对 → 自动定时 |
| Instagram | ✅ 全自动 | 🎬 视频 / 🖼️ 图片（轮播最多10张） | 设定时 → 自动提交；图片模式不触发 Reels 流程 |
| **B站** | ⚠️ 只填不发 | 🎬 视频 | 填好标题/封面/标签后停止，等你手动点「发布」；内置前置查重 |
| **抖音** | ⚠️ 只填不发 | 🎬 视频 / 🖼️ 图文 | 同上；图片模式自动切到「图文」tab |
| **小红书** | ⚠️ 只填不发 | 🎬 视频 / 🖼️ 图片 | 同上；images 替代 video 时走纯图文流程 |
| **视频号** | ⚠️ 只填不发 | 🎬 视频 | 同上（检测定时是否被平台篡改） |

> 国内四家"最后一下你点"是主动的风控设计，非功能缺失。

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
node src/hub.mjs review <id> [天数]  # 输出某期 N 天复盘（--notify 发飞书）
node src/hub.mjs weekly              # 周报（每周二自动发）
node src/hub.mjs daemon              # 守护进程：定时轮询 + 自动取链接 + 提醒
node src/hub.mjs notify-test         # 测试飞书通知是否正常

node src/web.mjs                     # 独立启动管理网页（8787 端口）
node src/demo.mjs                    # 演示模式（假数据，看效果）
```

---

## 已修复问题记录

### 🔴 P0 — 阻断性问题（已全部修复）

| 问题 | 影响 | 修复文件 | commit |
|------|------|---------|--------|
| `init.mjs` Chrome 检测硬编码 macOS 路径 | Windows 下报「没找到 Google Chrome」 | `src/init.mjs` | `26e6a26` |
| `hub.mjs` NAS 重挂用 macOS 专有命令 | Windows 下载不到 NAS 时脚本报错 | `src/hub.mjs` | `26e6a26` |
| `pruneLogs()` 90 天过滤形同虚设 | 旧日志全部保留，数据库无限增长 | `src/db.mjs` | `8c11e61` |
| `node src/web.mjs` 静默 exit 0 | Web UI 无法独立打开 | `src/web.mjs` | `8c11e61` |

### 🟡 P1/P2 — 中等优先级（已全部修复）

| 问题 | 影响 | 修复方式 | commit |
|------|------|---------|--------|
| `caffeinate` Windows ENOENT 崩溃 | X 长文模式下脚本异步崩溃 | macOS 用 caffeinate；Windows 直接 spawn node | `defaebe` |
| `busy()` 在 direct 模式恒 false | 无法阻止多期并发，可能同时触发两轮 Chrome 操作 | 实现 PID 锁文件方案 | `defaebe` |
| 飞书通知同一事件重复发送 | 同一完成事件收双通知 | 加显式 `_metaKey` 精确互斥 | `defaebe` |
| `board.mjs lark-cli` Windows ENOENT | 运行 `hub board` 直接崩溃 | 检测到 Windows 则 warn 并跳过 | `defaebe` |
| publisher 上传等待循环中标签页丢失卡死 | Chrome 崩溃后任务永远停「上传中」 | 每轮先验证 tab 存活 | `6d7d78f` |

### 🟢 增强项

| 项目 | 状态 | 说明 |
|------|------|------|
| 每日自动 pruneLogs（daemon 每周日清理） | ✅ 已实现 | |
| B站前置查重 | ✅ 已实现 | 调投稿管理 API |
| X / YouTube / Instagram 查重 | ✅ 已实现 | |
| E1 平台改版 DOM 告警 | ❌ 未实现 | |
| E2 Chrome profile 缓存清理 | ❌ 未实现 | |
| E3 ffprobe 视频格式校验 | ❌ 未实现 | |
| E4 Web UI 一键提交按钮 | ❌ 未实现 | |
| E5 多期并发队列 | 🟡 部分缓解 | PID 锁方案，单人本机可忽略 |

---

## 测试验证

```powershell
node test_e2e.mjs    # E2E 集成测试（Chrome CDP ↔ publish.js 协同）
```

E2E 测试结果（实测）：
- ✅ Chrome 启动成功（CDP 端口 9333 连通）
- ✅ 插件 lib/publish.js 加载成功（7 个平台适配器）
- ✅ tabs.create/get/query/remove 全流程正常
- ✅ preflight 登录检测可正常读取页面 DOM 状态
- ✅ pruneLogs 日期过滤逻辑正确（4000 条旧日志测试场景）

### 图片内容发布

publish.json 新增可选字段 `images`（相对路径数组），替代 `video` 走图片发布流程：

```json
{ "version": 1, "id": "ep-xxx", "images": ["图片/a.jpg", "图片/b.png"], "platforms": { "x": {"text": "..."}, "xhs": {"title":"...","body":"..."} } }
```

---

## License

MIT