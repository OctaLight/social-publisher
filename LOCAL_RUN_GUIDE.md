# 🌾 麦田工作台 · 本地运行指南

## 仓库关系说明

| 仓库 | 角色 | 是否独立可运行 |
|------|------|---------------|
| [mdldm-maitian](https://github.com/CzzzzzzJ/mdldm-maitian) | 中控 Hub：调度/防重/数据库/Web大屏/飞书通知 | ✅ 可以（配合插件） |
| [mdldm-maitian-publisher](https://github.com/CzzzzzzJ/mdldm-maitian-publisher) | Chrome 插件源码：各平台填写引擎 | ❌ 单独无意义，是 Hub 的依赖库 |

**两者不是重复，而是分层的父子依赖。** publisher 的 `lib/publish.js` 被 hub 动态加载。

---

## 本地运行方式（Windows 实测可用）

### 环境要求
- Node.js ≥ 22.5（用到内置 `node:sqlite`）
- Google Chrome（任何版本）
- 网络能访问 GitHub + 各大平台

### 一键安装
```powershell
# 1. 克隆主仓库
git clone https://github.com/CzzzzzzJ/mdldm-maitian.git maitian
cd maitian

# 2. 克隆发布助手（作为 lib 目录使用）
git clone https://github.com/CzzzzzzJ/mdldm-maitian-publisher.git ../maitian-publisher

# 3. 安装 ws 依赖（CDP WebSocket 用）
npm install ws

# 4. 生成配置
Copy-Item config.example.json config.json
# 编辑 config.json，确保以下项正确：
#   pluginDir: "D:/.../maitian-publisher"
#   chromePort: 9333
#   chromeProfile: "D:/.../maitian/chrome-profile"
#   rootPath: "<你的视频工程文件夹>"
```

### 启动步骤
```powershell
# 启动专用 Chrome（带调试端口）
node src/hub.mjs login
# → 会在专用 Chrome 里打开 7 个平台登录页

# 登录完各平台后，检查状态
node src/hub.mjs doctor

# 启动网页大屏
node src/demo.mjs           # 假数据演示
# 或
node src/web.mjs            # 真实数据（需配好 config）
# → 浏览器打开 http://localhost:8787/?key=<web.key>
```

---

## 本仓库对 direct 模式的修复

原 `chrome-shim.mjs` 有两个问题会导致直接模式崩溃：

| 问题 | 影响 | 修复 |
|------|------|------|
| `tabs.query()` 永远返回 `[]` | `publish.js` 的 `withFocus()` 取不到当前活动页，crash | 改为从 CDP `Target.getTargets` 实时获取 |
| `debugger.onDetach` 不触发 | 外部关闭标签页后内部状态不更新 | 已在 `Target.targetDestroyed` 里补清理逻辑 |
| `chrome.runtime.id` 缺失 | `removeForeignFrames` 无法判断哪些 iframe 是自己注入的 | 设为 `'publish-hub'`，并补充 `getManifest()` |

详见 `FUSION_ANALYSIS.md`。

---

## 两种运行模式对比

| 特性 | direct（直连） | plugin（桥接） |
|------|---------------|----------------|
| 是否需要装插件 | ❌ 不需要 | ✅ 需要（Chrome 商店版） |
| 稳定性 | ⚠️ 刚完成 patch，尚在验证 | ✅ 作者生产环境验证过 |
| 扩展性 | 同插件代码，随时跟上游 | 插件商店自动更新 |
| 推荐场景 | 开发/测试 / Windows 用户 | macOS 生产环境 |

**建议**：先用 `driver: "direct"` 跑通流程，确认一切正常后再考虑切换到 `plugin` 模式（装商店版插件）。

---

## 快速测试脚本

```powershell
cd D:\程序\xxai-feed\maitian
node test_e2e.mjs      # E2E 集成测试
node check_tabs.mjs    # 查看当前 Chrome 标签页
```

---

## 已知限制

1. **国内四家只填不发**：B站/抖音/小红书/视频号最后一下需要你手动点发布（风控设计）
2. **专用 Chrome 隔离**：必须用独立 profile，不能用日常浏览器的账号
3. **成片格式**：必须是 faststart（moov atom 在文件头），否则小红书会报错
4. **Node.js 版本**：必须 ≥ 22.5，旧版没有内置 SQLite
5. **Windows 未官方支持**：README 标注 macOS-only，但经实测 Windows 下 core 功能可用
