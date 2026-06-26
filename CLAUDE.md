# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Chrome 浏览器插件（Manifest V3），用于将饭否（fanfou.com）推文下载为 Markdown 或 HTML 格式。无构建步骤、无依赖——直接加载为 Chrome 已解压扩展即可运行。

## 架构与数据流

整个应用逻辑集中在 `popup.js` 一个文件中（~396 行），无模块化、无 background script、无 content script 声明。

**执行流程：**
1. 用户点击插件图标 → 弹出 `popup.html`（320px 宽 popup）
2. 用户选择下载范围（全部/最近 N 条）和格式（Markdown/HTML）
3. `popup.js` 通过 `chrome.tabs.query()` 获取当前标签页，校验是否在 `fanfou.com`
4. 从 URL 路径提取用户 ID（如 `fanfou.com/yihan1944`），排除 `home`/`login`/`settings` 等保留路径
5. **第 1 页**：`chrome.scripting.executeScript()` 注入 `fetchCurrentPageTweets()` 到活动标签页，直接读取 DOM
6. **后续页**：`fetch()` + `credentials: 'include'` 请求 `https://fanfou.com/{userId}/p.{N}`，用 `DOMParser` 解析 HTML
7. 翻页直到达到目标数量或无下一页链接
8. 生成 Markdown/HTML 内容，通过 `chrome.downloads.download()` + `data:` URL + `saveAs: true` 触发下载

**Fanfou DOM 选择器（关键依赖）：**
- `#stream ol li` — 单条推文容器
- `#stream span.content` — 推文文本
- `#stream a.time` — 推文时间（取 `title` 属性获取完整时间）
- `.paginator` — 翻页区域，"下一页"链接判断是否还有后续页
- `#sidebar .vcard a[href*="/friends/"]` / `#user_stats a[href*="/friends/"]` — 从侧边栏提取用户 ID

## Chrome API 使用

| API | 用途 |
|---|---|
| `chrome.tabs.query()` | 获取活动标签页 ID 和 URL |
| `chrome.scripting.executeScript()` | 注入抓取函数到饭否页面 |
| `chrome.downloads.download()` | 触发文件下载 |

## 打包发布

无构建步骤。手动打包为 zip 上传 Chrome Web Store：

```bash
zip -r get-fan.zip manifest.json popup.html popup.css popup.js icons/
```

## 已知问题

- `popup.js` 中 `loadScript()` 函数（362-369 行）已定义但从未调用——死代码
- manifest 只声明了 128px 图标，`icons/README.md` 提到的 16px/48px 变体不存在
- HTML 格式单选按钮的 value 是 `pdf`，实际生成的是 HTML（用户可 `Ctrl+P` 打印为 PDF）
- 无 Chrome storage API 使用——每次打开 popup 不保留上次设置

## 语言约定

文档和注释使用中文，技术术语保留英文。
