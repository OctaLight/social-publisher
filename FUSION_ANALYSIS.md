# mdldm-maitian 与 maitian-publisher 融合分析

## 一、两个仓库的关系（不是重复，是分离设计）

```
mdldm-maitian（中控 Hub）        mdldm-maitian-publisher（Chrome插件）
├── hub.mjs          ← 调度/防重/数据采 ─┐
├── driver.mjs       ← 驱动选择器        │ 直接加载 lib/publish.js
├── chrome-shim.mjs  ← CDP 模拟层       ├─────▶  publish.js（核心发布逻辑）
├── db.mjs           ← SQLite 数据库    │         ├── cdp.js     （CDP 操作封装）
├── web.mjs          ← 管理网页         │         ├── platforms/*.js（7个平台适配器）
└── skills/          ← AI 文案 skill   │         └── bundle.js  （校验规则）
                                     │
└── config.json 指定 pluginDir ─────────┘
          指向 publisher 源码目录
```

**publisher 的 `lib/` 代码被 hub 动态 import（`pathToFileURL`），两种模式共用同一份：**

| 模式 | 如何运行 publisher 代码 | 说明 |
|------|------------------------|------|
| `driver: "direct"` (默认) | Node.js 直接 import，用 chrome-shim 提供 chrome.* API | ✅ 可本地跑，无插件依赖 |
| `driver: "plugin"` | 通过 DevBridge HTTP 向插件面板发消息 | 需要日常 Chrome 里装好插件 |

## 二、当前 direct 模式的已有问题（补丁清单）

### ❌ P0: `tabs.query()` 返回空数组
`publish.js` 第 81 行：
```js
const [prev] = await chrome.tabs.query({ active: true, windowId: tab.windowId });
await chrome.tabs.update(prev.id, { active: true }); // ← prev 是 undefined → crash
```
影响：任何用到 `withFocus()` 的平台（B站封面设置等）会崩溃。

### ❌ P1: `debugger.onDetach` 未触发
外部关闭标签页时，shim 只处理了 `Target.targetDestroyed` 事件，但 `cdp.js` 维护了一个 `attached` Set，不更新会导致后续 sendCommand 报 "already detached" 错误。

### ⚠️ P2: `scripting.executeScript` 抛异常
`cdp.js removeForeignFrames()` 在直连模式下会被 `send()` 里的 catch 捕获并重试一次，不会崩，但功能缺失——YouTube 有 YouMind 扩展时无法清理 foreign iframe。

### ✅ 已正确实现
- `chrome.tabs.create/get/update/remove` — 完整
- `chrome.debugger.attach/detach/sendCommand` — 完整（经 WebSocket→CDP）
- `Runtime.evaluate` / `DOM.setFileInputFiles` / `Input.dispatch*` — 全部可用

## 三、融合方案

### 方案 A：软链接（推荐，最小改动）
把 publisher 做成 maitian 的子目录（git submodule 或手动复制），config 中指向自身。

### 方案 B：合并为一个仓库
把 publisher 的 `lib/` 合并进 maitian 的 `src/plugin-lib/`，移除外部依赖。

**本次执行方案 A + 打补丁：**
1. 在 `maitian/src/chrome-shim.mjs` 补全 `tabs.query()` 和 `onDetach`
2. 确认现有配置能跑通完整链路
