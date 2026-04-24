---
name: wechat-article-archive
description: >
  长文章与长文本归档助手。用户发送任意长文章链接、微信公众号文章、知乎文章/回答、博客、
  新闻、网页长文，或直接粘贴一大段文字并要求保存、归档、收藏、存到 Obsidian、做文章记录时，
  自动抓取或整理正文、梳理总结核心要点，并将链接/原文信息和摘要保存到 Obsidian Vault 的
  「文章记录」文件夹中。重点支持 mp.weixin.qq.com、zhihu.com、zhuanlan.zhihu.com，也支持
  其他可访问网页链接和无链接的长文本。即使用户只发送一个疑似文章链接，或粘贴超过约 500 字
  的长文本并表达保存/整理意图，也应触发此 skill。
compatibility:
  tools: [WebFetch, Read, Write, Bash, Glob]
  skills: [web-access]
  notes: >
    优先使用 WebFetch/Jina 等轻量方式处理普通网页；微信、知乎、登录态页面或动态页面优先使用
    web-access skill 的 CDP 浏览器能力。
    如果 web-access 不可用，需要文件系统访问权限保存到 Obsidian Vault。
---

# 长文章与长文本归档

接收任意长文章链接、微信公众号、知乎文章/回答或用户直接粘贴的长文本，自动完成抓取/整理、总结、归档到 Obsidian Vault 的全流程。

---

## 工作流程

### 1. 接收输入
用户输入可能是：
- `https://mp.weixin.qq.com/s/...`
- `https://mp.weixin.qq.com/s?__biz=...&mid=...`
- `https://zhuanlan.zhihu.com/p/...`
- `https://www.zhihu.com/question/.../answer/...`
- `https://www.zhihu.com/pin/...`
- 其他博客、新闻、研究文章、长网页链接
- 短链接或带追踪参数的链接
- 用户直接粘贴的大段文字、访谈摘录、邮件、备忘录或文章草稿

先判断输入类型：
- **有 URL**：归档为网页文章；保留原始 URL。
- **无 URL 但有长文本**：归档为用户提供文本；`url` 留空或写 `manual-input`。
- **既有 URL 又有补充文字**：优先抓取 URL 正文，并把用户补充文字纳入"补充材料"或摘要判断。

### 2. 抓取文章内容

按平台选择最小可行抓取方式。普通网页优先使用轻量抓取；微信、知乎、登录态页面或动态页面使用 CDP，并复用用户已有 Chrome 登录态。

### 2.0 CDP 连接复用原则

不要把 CDP 当成每次任务都要重新构建的临时资源。优先复用已经运行的 Chrome remote debugging server 和 web-access proxy：

1. 先检查 CDP/proxy 是否已经可用：
   - `node "$HOME/.codex/skills/web-access/scripts/check-deps.mjs"`（Codex）
   - `node "$HOME/.claude/skills/web-access/scripts/check-deps.mjs"`（Claude Code）
   - 或直接尝试 `curl -s http://localhost:3456/targets`
2. 如果 `localhost:3456` proxy 已经有响应，直接创建后台 tab 抓取文章，不要重启 Chrome，不要要求用户重新开启 CDP。
3. 如果 proxy 不存在但 Chrome remote debugging 已开启，运行 `check-deps.mjs` 让它连接现有 Chrome。
4. 如果确认 Chrome 没有开启 remote debugging，立即停止抓取/归档流程，提示用户打开：
   - `chrome://inspect/#remote-debugging`
   - 勾选 `Allow remote debugging for this browser instance`
   - 如果 Chrome 提示 `DevTools remote debugging requires a non-default data directory`，说明新版 Chrome 不允许默认资料目录直接开 CDP。请用户用独立资料目录启动：
     ```bash
     /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
       --remote-debugging-port=9222 \
       --user-data-dir="$HOME/.codex/chrome-cdp-profile"
     ```
     并提醒用户：这是独立 Chrome 资料目录，首次使用可能需要在这个窗口里重新登录知乎、微信等网站。
   不要继续尝试其他抓取、归档或替代方案，等用户明确说已开启后再继续。
5. 任务完成后关闭自己创建的 tab，保留 proxy 和用户原有 Chrome 标签页。

**CDP 故障判断**：
- `curl -s http://127.0.0.1:9222/json/version` 能返回 JSON，才说明 remote debugging 真的可用；不要只看 Chrome 是否打开或 `chrome://inspect` 页面是否还停留在旧文字。
- 如果 `127.0.0.1:9222` 可用但 `localhost:3456/targets` 失败，通常是旧的 web-access proxy 占用 `3456`，或 proxy 读到了过期的 `DevToolsActivePort`。先重启 proxy，再运行 `check-deps.mjs`。
- `DevToolsActivePort` 文件可能残留旧 WebSocket 路径。判断当前连接时，以 `/json/version` 返回的 `webSocketDebuggerUrl` 为准。

**如果 `web-access` skill 可用**：
1. 只有在需要浏览器登录态、动态渲染、反爬绕过或 WebFetch 失败时，加载 `web-access` skill。
2. 使用其 CDP 模式创建新 tab 访问文章链接。
3. 根据域名选择提取策略。
4. 提取完成后关闭 tab。

**普通网页提取策略**：
1. 优先尝试 `WebFetch`、Jina 或页面公开内容提取。
2. 如果抓取结果只有导航、广告、登录提示或正文明显缺失，再切换 CDP。
3. 提取字段：
   - 标题：`h1`、`meta[property="og:title"]`、`document.title`
   - 作者：`author` meta、页面作者区、署名文本
   - 日期：`time` 标签、`article:published_time`、正文附近日期
   - 正文：`article`、`main`、`[role="main"]`、正文密度最高的文本容器

**微信公众号提取字段**：
通过 `/eval` 提取页面内容：
   - 文章标题：`document.querySelector('#activity_name')?.textContent.trim()` 或 `document.title`
   - 公众号名称：`document.querySelector('#js_name')?.textContent.trim()`
   - 发布日期：`document.querySelector('#publish_time')?.textContent.trim()`
   - 正文内容：`document.querySelector('#js_content')?.innerText`

**知乎文章/回答提取字段**：
先滚动页面触发正文加载，再通过 `/eval` 提取：
   - 标题：
     - 专栏文章：`.Post-Title`、`h1.Post-Title`、`document.title`
     - 问答回答：`.QuestionHeader-title`、`h1`、`document.title`
   - 作者：`.AuthorInfo-name`、`.UserLink-link`、`.ContentItem-author`
   - 发布时间：`time` 标签、`.ContentItem-time`、`.ContentItem-time a`
   - 正文：
     - 专栏文章：`.Post-RichText`、`.RichText`
     - 回答：`.QuestionAnswer-content .RichContent-inner`、`.RichContent-inner`、`.RichText`
   - 点赞/赞同数（如页面可见）：`.VoteButton`、`.ContentItem-actions button`

如果知乎页面打开后出现登录墙、展开按钮或"阅读全文"，优先在当前 tab 内点击可见按钮再提取。用户已登录时，尽量使用其现有登录态，不要改动账号设置或发起互动。

**如果 `web-access` skill 不可用或 CDP 失败**：
- 对普通网页先尝试 `WebFetch`、Jina 或公开网页抓取
- 对知乎链接可尝试 `WebFetch` 或公开网页抓取作为备选
- 对微信文章可尝试 `WebFetch` 作为备选（但微信文章大概率会触发 CAPTCHA）
- 如果仍失败，请用户复制粘贴文章内容，或提供文章标题和关键信息

**提取目标字段**：
- **文章标题**
- **发布日期**（如有）
- **作者/公众号名称/知乎作者**（如有）
- **平台/来源类型**（微信公众号、知乎专栏、知乎回答、网页文章、用户粘贴文本等）
- **正文内容**
- **原始 URL**（如有）
- **关键词**：提取 5-10 个适合 Obsidian 检索和双链组织的关键词，优先使用具体概念、人物、机构、产品、理论、事件和行业词，避免泛词。

### 2.5 处理用户直接粘贴的长文本

如果用户没有提供 URL，但提供了大段文字：
1. 从文本中识别标题；如果没有明确标题，生成一个简短、具体的标题。
2. 将来源标记为 `用户提供文本`，平台标记为 `手动输入`。
3. 保留原始文本的主要结构，不要过度改写成另一篇文章。
4. 摘要应忠于原文，不添加未经原文支持的观点。
5. 如果文本明显是转写稿、访谈、课程笔记或草稿，标签中体现对应类型。

### 3. 梳理总结
基于抓取到的内容，生成结构化摘要：
- 核心观点（3-5 条 bullet points）
- 关键论据/数据
- 关键词（5-10 个，用于 Obsidian 检索、标签和双链）
- 文章类型标签（如：行业分析、产品评测、观点评论、教程指南、新闻资讯等）

### 4. 保存到 Obsidian Vault

**目标路径**：`~/Documents/Obsidian Vault/文章记录/`

**文件命名格式**：`YYYY-MM-DD_文章标题.md`

文件名和 frontmatter 中的 `date` 使用**保存当天日期**，不是文章发布时间。文章原始发布时间单独写入 `published` 字段和正文元信息。

如果同一天有多篇同名文章，在文件名后加 `_1`、`_2` 等后缀。

**文件模板**：
```markdown
---
title: 文章标题
url: 原文链接
date: YYYY-MM-DD
published: YYYY-MM-DD（如有）
source: 公众号名称或知乎作者
platform: 微信公众号/知乎专栏/知乎回答/网页文章/手动输入
author: 作者（如有）
tags: [标签1, 标签2]
keywords: [关键词1, 关键词2, 关键词3]
---

# 文章标题

**原文链接**：[链接](URL)

**来源**：公众号名称、知乎作者、网站名称或用户提供文本

**平台**：微信公众号/知乎专栏/知乎回答/网页文章/手动输入

**保存时间**：YYYY-MM-DD

**原文发布时间**：YYYY-MM-DD（如有）

---

## 核心摘要

- 要点1
- 要点2
- 要点3

## 关键词

关键词1、关键词2、关键词3

## 详细内容

（文章正文或更详细的梳理）

## 个人备注

（留白供用户后续补充）
```

### 5. 向用户确认
保存完成后，向用户汇报：
- 文章标题
- 保存路径
- 核心摘要预览

---

## 边界情况处理

| 情况 | 处理方式 |
|------|---------|
| web-access skill 不可用 | 知乎先尝试 WebFetch；微信尝试 WebFetch，失败则请用户复制粘贴文章内容 |
| CDP/proxy 未连接 | 先尝试复用或启动 web-access proxy；如果 Chrome remote debugging 未开启，立即提示用户开启并停止本次流程 |
| Chrome 要求 non-default data directory | 提示用户用 `--remote-debugging-port=9222 --user-data-dir="$HOME/.codex/chrome-cdp-profile"` 启动；说明这是独立资料目录，首次使用可能需要重新登录，但后续会保留登录态 |
| 9222 可用但 3456 不可用 | 重启 web-access proxy；检查是否有旧 proxy 占用端口，连接判断以 `/json/version` 的当前 WebSocket 地址为准 |
| CDP 抓取失败/CAPTCHA | 请用户复制粘贴文章内容，或提供文章标题+核心内容 |
| 链接无法访问（参数错误/404） | 请用户确认链接是否可在原平台打开，或重新分享链接 |
| 文章已存在 | 询问用户是否覆盖、重命名或跳过 |
| 非微信/知乎文章链接 | 按通用网页文章处理；如果正文抓不到，再请用户粘贴正文 |
| 用户只发链接无指令 | 自动识别并执行归档流程 |
| 用户粘贴大段文字 | 按手动输入归档，生成标题、摘要、标签并保存 |
| 用户提供了标题和部分内容 | 基于提供的信息生成摘要并保存，原文链接留空或标注 `manual-input` |
| Vault 路径不存在 | 尝试创建路径，若失败则询问用户正确路径 |
| 文章发布时间和保存时间不同 | 文件名和 `date` 使用保存时间；文章发布时间写入 `published` 与正文元信息 |

---

## 运行规则

1. **自动触发**：检测到 `mp.weixin.qq.com`、`zhihu.com`、`zhuanlan.zhihu.com` 域名链接，或用户明确要求归档任意长文章链接/大段文字时自动执行归档流程
2. **语言对齐**：总结使用与文章相同的语言
3. **不输出思考过程**：直接执行，完成后汇报结果
4. **尊重用户空间**：不在文件中添加 AI 生成的个人评论，备注栏留白
5. **文件安全**：写入前检查是否已存在同名文件，避免意外覆盖
6. **复用浏览器连接**：不要每次重新构建 CDP；优先复用现有 Chrome remote debugging 与 web-access proxy
7. **缺少 remote debugging 即停止**：如果 Chrome 未开启 remote debugging，只提醒用户开启，不继续抓取、不继续归档
8. **忠于原文**：对长文本和网页正文只做摘要、结构化和标签，不编造来源、作者、日期或数据
9. **保存时间优先**：文件名、标题中的日期和 frontmatter `date` 均使用保存文章的日期；原文写作/发布时间只放在 `published` 字段和正文元信息中
10. **关键词优先**：每篇归档都提取 5-10 个关键词，便于 Obsidian 标签、搜索和双链组织
