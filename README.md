# GitHub Digger

A Claude Code skill for identifying Chinese contributors from GitHub repositories — for talent recruiting and sourcing.

## What it does

Given a GitHub repo (or a topic keyword), GitHub Digger:

1. **Auto-detects repo type** — community open source (PR mode) vs enterprise internal research repo (Commit mode)
2. **Scans contributors** — from closed PRs or commit history depending on repo type
3. **Traces arXiv papers** — for research repos, fetches the full paper author list to surface people who don't push code but own the architecture
4. **Enriches profiles** — name, company, location, email from 4 sources (commit metadata, paper footnotes, GitHub profile, personal homepage)
5. **Scores Chinese identity** — high / medium confidence based on name, location, company signals
6. **Outputs a ready-to-use recruiting list** — with personalized outreach templates

## Usage with Claude Code

```
/github-digger ray-project/ray
/github-digger ByteDance-Seed/VideoWorld role: Data Infra Engineer
/github-digger "AI native data infra repos with Chinese contributors"
```

## Modes

| Mode | Trigger | Strategy |
|---|---|---|
| **Repo Discovery** | Topic/keyword input | WebSearch → ranked repo list → drill down |
| **PR Mode** | Community repo (≥10 closed PRs) | Scan closed PRs |
| **Commit Mode** | Enterprise/research repo (<10 PRs) | Scan commits + arXiv paper tracing |

## Email sources (in priority order)

1. **Commit metadata** — highest yield for enterprise repos (e.g. `renzhongwei@bytedance.com`)
2. **Paper corresponding author footnote** — research repos often list email in arXiv PDF
3. **GitHub profile field** — when explicitly set public
4. **Personal homepage** — scan for `mailto:` links

## Install

Place `SKILL.md` in your Claude skills directory:

```
~/.claude/skills/github-digger/SKILL.md
```

## Changelog

### v2.0 (2025-04-10)
- **Commit Mode**: auto-switch when PR count < 10 (enterprise repo detection)
- **arXiv paper tracing**: surfaces full author lists from README arxiv links
- **4-source email strategy**: added paper footnote as a new email source
- **Repo Discovery Mode**: accepts topic/keyword, finds repos before scanning
- **Python urllib preferred** over WebFetch (avoids enterprise network blocks)
- Output enhancements: scan strategy, email source, `[commit]`/`[paper]`/`[both]` tagging

### v1.0 (initial)
- PR-based contributor scanning
- Chinese identity scoring
- Profile enrichment with 2-source email

## Contributors

| Name | Role |
|---|---|
| [@fflashxu](https://github.com/fflashxu) | Creator & Maintainer |
| [Claude Sonnet 4](https://claude.ai) | Skill design & implementation |
