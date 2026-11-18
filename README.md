# xxai_feed_maitian（麦田）本地部署版

基于 [CzzzzzzJ/mdldm-maitian](https://github.com/CzzzzzzJ/mdldm-maitian) 的本地化改造版本，适配 Windows 直连运行。

## 仓库说明

| 原始仓库 | 角色 | 关系 |
|---------|------|------|
| [mdldm-maitian](https://github.com/CzzzzzzJ/mdldm-maitian) | 中控 Hub（调度/防重/数据/Web大屏/飞书通知） | 主仓库，本版本在此基础上增加 Windows 支持 |
| [mdldm-maitian-publisher](https://github.com/CzzzzzzJ/mdldm-maitian-publisher) | Chrome 插件源码（各平台填写引擎） | 作为依赖目录 `maitian-publisher/` 存在 |

**两个仓库不是重复，而是分层设计：** publisher 的 `lib/publish.js` 被 hub 动态加载，共享同一份适配器代码。

---

## 本次改造内容

### 修复 direct 模式的 Bug

原 `src/chrome-shim.mjs` 在 Node.js 直连模式下有两个导致崩溃的问题：

| # | 问题 | 影响 | 修复方式 |
|---|------|------|---------|
| P0 | `tabs.query()` 永远返回空数组 `[]` | `publish.js` 的 `withFocus()` 取不到当前活动页，crash | 改为从 CDP `Target.getTargets` 实时枚举已管理标签 |
| P1 | `debugger.onDetach` 监听器不触发 | 外部关闭标签页后内部 `tabs/byTarget` Map 泄漏 | 在 `Target.targetDestroyed` 事件里补清理逻辑 |
| P2 | `chrome.runtime.id` 缺失 | `removeForeignFrames`（YouTube 清理 foreign iframe）无法判断来源 | 补充 `runtime.id = 'publish-hub'` 及 `getManifest()` |
| P3 | `detach({ tabId })` 未清空 sessionId | 重用已 detach 的 tabId 会导致 Session not found 错误 | 移除 sessionId 引用 |

### Windows 兼容增强

- `config.json` 默认使用 `driver: "direct"` + `pluginDir` 指向本地源码，无需安装 Chrome 商店插件
- 安装了 `ws` 包用于 CDP WebSocket 通信（Node.js 内置 WebSocket 在部分场景不稳定）
- 专用 Chrome profile 路径默认为项目内 `chrome-profile/` 子目录

---

## 快速启动

```powershell
# 1. 确保环境
node -v   # 需要 ≥ 22.5
# git clone 已在本地完成：D:\程序\xxai-feed\maitian\

# 2. 启动专用 Chrome（带调试端口）并打开各平台登录页
cd D:\程序\xxai-feed\maitian
node src/hub.mjs login
# → 弹出专用 Chrome，里面打开了 B站/抖音/小红书/视频号/YouTube/X/Instagram 登录页
# → 在各页面完成扫码或账号密码登录

# 3. 体检确认各平台状态
node src/hub.mjs doctor

# 4. 启动管理网页
node src/web.mjs
# → 浏览器打开 http://localhost:8787/?key=<web.key>
```

### 配置要点（config.json）

```json
{
  "plugin": "local",
  "pluginDir": "D:/程序/xxai-feed/maitian-publisher",
  "driver": "direct",
  "chromePort": 9333,
  "chromeProfile": "D:/程序/xxai-feed/maitian/chrome-profile",
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

| 平台 | 自动化程度 | 说明 |
|------|-----------|------|
| X (Twitter) | ✅ 全自动 | 填文案 → 自动点 Schedule |
| YouTube | ✅ 全自动 | 查 Studio 列表核对 → 自动定时 |
| Instagram | ✅ 全自动 | 设定时 → 自动提交 |
| **B站** | ⚠️ 只填不发 | 填好标题/封面/标签后停止，等你手动点「发布」 |
| **抖音** | ⚠️ 只填不发 | 同上 |
| **小红书** | ⚠️ 只填不发 | 同上（最保守策略，不碰内部接口） |
| **视频号** | ⚠️ 只填不发 | 同上（还会检测定时是否被平台篡改） |

> 国内四家"最后一下你点"是作者主动的风控设计，非功能缺失。

---

## 命令参考

```bash
node src/hub.mjs login          # 打开专用 Chrome + 各平台登录页
node src/hub.mjs doctor         # 体检：检查各平台登录态和上传页结构
node src/hub.mjs plan <目录>    # 登记分发计划（不上传）
node src/hub.mjs run <目录>     # 执行发布流程
node src/hub.mjs status [id]    # 查看发布进度
node src/hub.mjs sync <id>      # 取回已发布链接
node src/hub.mjs metrics [id]   # 采集流量数据
node src/web.mjs                # 启动管理网页（8787 端口）
node src/demo.mjs               # 演示模式（假数据，看效果）
```

---

## 测试验证

```powershell
cd D:\程序\xxai-feed\maitian
node test_e2e.mjs    # E2E 集成测试（Chrome CDP ↔ publish.js 协同）
```

E2E 测试结果（实测）：
- ✅ Chrome 启动成功（CDP 端口 9333 连通）
- ✅ 插件 lib/publish.js 加载成功（7 个平台适配器：bilibili/xhs/douyin/channels/youtube/x/instagram）
- ✅ tabs.create/get/query/remove 全流程正常
- ✅ preflight 登录检测可正常读取页面 DOM 状态

---

## License

MIT（继承上游）
