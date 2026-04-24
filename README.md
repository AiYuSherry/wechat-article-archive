# wechat-article-archive

A Codex / Claude Code skill for archiving long-form Chinese articles and pasted long text into an Obsidian vault.

Despite the legacy name, this skill is no longer limited to WeChat articles. It supports:

- WeChat official account articles (`mp.weixin.qq.com`)
- Zhihu columns and answers (`zhihu.com`, `zhuanlan.zhihu.com`)
- General long-form web articles, blogs, essays, and news pages
- Manually pasted long text

The skill extracts article metadata, summarizes key points, generates Obsidian-friendly keywords, and saves a Markdown note under `文章记录`.

## Prerequisites

This skill depends on the `web-access` skill for browser-based CDP access when pages require login state, dynamic rendering, or anti-bot handling.

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

## Installation

For Codex:

```bash
mkdir -p ~/.codex/skills
git clone https://github.com/AiYuSherry/wechat-article-archive.git ~/.codex/skills/wechat-article-archive
```

For Claude Code:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/AiYuSherry/wechat-article-archive.git ~/.claude/skills/wechat-article-archive
```

## Usage

Send a supported article link or paste a long text block and ask the assistant to archive it:

```text
帮我归档这篇文章：https://zhuanlan.zhihu.com/p/...
```

```text
把下面这段长文存到 Obsidian 文章记录里：
...
```

The generated note uses the archive date as the filename date and frontmatter `date`. The original article publication time is stored separately in `published`.

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

## Notes

- If Chrome remote debugging is not enabled, the skill should stop and ask the user to enable it rather than continuing with unreliable fallbacks.
- If `127.0.0.1:9222/json/version` works but `localhost:3456/targets` fails, Chrome CDP is available and the issue is likely the `web-access` proxy layer.
- The skill intentionally leaves `## 个人备注` blank for the user.

## Acknowledgements

Thanks to 一泽Eze, the author of `web-access`, whose CDP browser access workflow makes reliable logged-in and dynamic-page article extraction possible.

## License

MIT
