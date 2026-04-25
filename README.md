# obsidian-content-archive

A Codex / Claude Code skill for archiving long-form articles, web pages, and pasted long text into an Obsidian vault.

一个用于将长文章、网页内容和粘贴的长文本归档到 Obsidian 的 Codex / Claude Code skill。

## Features / 功能

- WeChat official account articles (`mp.weixin.qq.com`)
- Zhihu columns and answers (`zhihu.com`, `zhuanlan.zhihu.com`)
- General long-form web articles, blogs, essays, news pages, and other readable URLs
- Manually pasted long text

- 微信公众号文章（`mp.weixin.qq.com`）
- 知乎专栏与回答（`zhihu.com`、`zhuanlan.zhihu.com`）
- 其他可访问的长网页、博客、随笔、新闻和文章链接
- 用户直接粘贴的长文本

The skill extracts article metadata, summarizes key points, generates Obsidian-friendly keywords, and saves a Markdown note under `文章记录`.

它会提取文章元数据、总结核心要点、生成适合 Obsidian 双链和检索的关键词，并把 Markdown 笔记保存到 `文章记录` 文件夹。

## Prerequisites / 前提条件

This skill depends on the `web-access` skill for browser-based CDP access when pages require login state, dynamic rendering, or anti-bot handling.

本 skill 依赖 `web-access`。当页面需要登录态、动态渲染或存在反爬限制时，会通过 `web-access` 使用浏览器 CDP 访问。

Install or make available:

- `web-access` skill
- Node.js 22 or newer
- Google Chrome with remote debugging enabled when CDP is needed
- An Obsidian vault, default target: `~/Documents/Obsidian Vault/文章记录/`

For newer Chrome versions, remote debugging may require a non-default profile directory:

```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$HOME/.codex/chrome-cdp-profile"
```

Using a persistent profile directory lets login state survive across sessions.

使用固定的 profile 目录可以让登录态在多次会话之间保留。

## Installation / 安装

For Codex:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/AiYuSherry/obsidian-content-archive.git ~/.codex/skills/obsidian-content-archive
```

For Claude Code:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/AiYuSherry/obsidian-content-archive.git ~/.claude/skills/obsidian-content-archive
```

## Usage / 使用

Send a supported article link or paste a long text block and ask the assistant to archive it:

发送文章链接，或粘贴一段长文本，然后让助手归档：

```text
帮我归档这篇文章：https://zhuanlan.zhihu.com/p/...
```

```text
把下面这段长文存到 Obsidian 文章记录里：
...
```

The generated note uses the archive date as the filename date and frontmatter `date`. The original article publication time is stored separately in `published`.

生成的笔记会用“归档当天”作为文件名日期和 frontmatter 里的 `date`；文章原始发布时间会单独记录在 `published` 字段里。

Example frontmatter:

```yaml
---
title: 文章标题
url: 原文链接
date: 2026-04-25
published: 2024-09-12
source: 来源名称
platform: 网页文章
author: 作者
tags: [观点评论, 科技]
keywords: [关键词1, 关键词2, 关键词3]
---
```

## Notes / 说明

- If Chrome remote debugging is not enabled, the skill should stop and ask the user to enable it rather than continuing with unreliable fallbacks.
- If `127.0.0.1:9222/json/version` works but `localhost:3456/targets` fails, Chrome CDP is available and the issue is likely the `web-access` proxy layer.
- The skill intentionally leaves `## 个人备注` blank for the user.

- 如果 Chrome remote debugging 没有开启，skill 应该停止并提醒用户开启，而不是继续使用不稳定的替代方案。
- 如果 `127.0.0.1:9222/json/version` 可访问，但 `localhost:3456/targets` 失败，通常说明 Chrome CDP 已可用，问题更可能出在 `web-access` 代理层。
- skill 会故意保留空白的 `## 个人备注`，方便用户自己补充。

## Acknowledgements / 致谢

Thanks to 一泽Eze, the author of `web-access`, whose CDP browser access workflow makes reliable logged-in and dynamic-page article extraction possible.

感谢 `web-access` 的作者一泽Eze。这个 skill 的登录态页面和动态页面抓取能力，建立在 `web-access` 提供的 CDP 浏览器访问工作流之上。

## License / 许可证

MIT
