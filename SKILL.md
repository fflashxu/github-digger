---
name: github-digger
description: >
  Find Chinese contributors from GitHub open source projects for talent recruiting and sourcing.
  Use this skill whenever the user wants to find Chinese engineers or developers from a GitHub repo,
  scan open source project contributors for recruiting, identify Chinese talent from PR history,
  or source candidates from projects like Ray, vLLM, LLaMA, PyTorch, or any GitHub repository.

  Triggers on phrases like: "找华人贡献者", "扫描 GitHub 华人", "find Chinese contributors from",
  "scan {repo} for Chinese engineers", "挖角 GitHub", "从 {repo} 找华人", "/github-digger {repo}",
  or any request to find/identify/list people from a specific GitHub project for recruiting purposes.

  Always invoke this skill when the user provides a GitHub repo and asks about contributors,
  engineers, or potential candidates — even if they don't say "Chinese" explicitly.
---

# GitHub Digger

Identify Chinese contributors from a GitHub repo's closed PRs, enrich their profiles with contact
info, and produce a ready-to-use recruiting candidate list.

## Inputs

- **repo** (required): GitHub repo in `owner/repo` format, full GitHub URL, **or a topic/keyword**
  when the user doesn't have a specific repo in mind (e.g. "AI data infra repos with Chinese contributors")
- **limit** (optional): How many recent closed PRs or commits to scan. Default: 100
- **role** (optional): The role you're recruiting for. Default: "Data Infra Tech Lead"

If the user provides a full URL like `https://github.com/ray-project/ray`, extract `ray-project/ray`.
If only a repo name is given (e.g., "vllm"), ask the user to confirm the owner before proceeding.
**If a topic/keyword is given instead of a repo**, switch to Repo Discovery Mode (see below).

## API Access

**Always prefer Python urllib over WebFetch for GitHub API calls.** Use this pattern:

```python
import urllib.request, json

def gh_get(path):
    url = f"https://api.github.com{path}"
    req = urllib.request.Request(url, headers={
        'User-Agent': 'Mozilla/5.0',
        'Accept': 'application/vnd.github+json'
    })
    with urllib.request.urlopen(req, timeout=15) as r:
        return json.loads(r.read())
```

**Rate limit handling**: If you get a 403 or `HTTP Error 403`, stop and tell the user:
> GitHub API rate limit reached (60 req/hour without authentication). Add a token at
> https://github.com/settings/tokens and pass it to me as "use token: ghp_xxx" to continue.

---

## Mode A: Repo Discovery Mode

**Trigger**: User provides a topic/keyword instead of a specific repo (e.g. "LLM training data infra").

1. Use **WebSearch** with multiple queries to find candidate repos matching the topic
2. Filter: Stars > 100 · Created/active in last 5 years · Chinese contributor signals in description/org
3. Return a ranked table of repos, then ask: "要对哪个 repo 做精细扫描？"
4. Continue to Mode B with the selected repo.

---

## Mode B: Repo Scan Mode

### Step 0: Detect repo type — choose scan strategy

First, quickly check PR count and commit pattern to classify the repo:

```python
pulls = gh_get(f"/repos/{owner}/{repo}/pulls?state=closed&per_page=5")
pr_count = len(pulls)
```

| PR count | Repo type | Strategy |
|---|---|---|
| ≥ 10 | Community open source | **PR mode** (original Step 1) |
| < 10 | Enterprise internal / research repo | **Commit mode** (Step 1B) |

**Enterprise research repo signals** (switch to Commit mode if 2+ match):
- Fewer than 10 closed PRs
- Commits go directly to main without PR
- Org name is a company (`ByteDance-Seed`, `KwaiVGI`, `OpenDriveLab`, etc.)
- README contains an arXiv link (`arxiv.org/abs/`)

---

### Step 1A: PR Mode — Fetch closed PRs (community repos)

```
GET /repos/{owner}/{repo}/pulls?state=closed&sort=updated&direction=desc&per_page=100
```

Collect unique contributor `login` values from `pull.user.login`. Skip bots (logins containing
`[bot]` or ending in `-bot`).

---

### Step 1B: Commit Mode — Fetch commits + arXiv authors (enterprise/research repos)

**1B-1: Scan commits**
```
GET /repos/{owner}/{repo}/commits?per_page=100
```
For each commit, extract: `author.login`, `commit.author.name`, `commit.author.email`, `commit.author.date`.

Collect unique logins + their commit count, latest commit date, and commit email.

**1B-2: arXiv paper tracing (critical for research repos)**

Fetch the README and scan for `arxiv.org/abs/` URLs:
```
GET /repos/{owner}/{repo}/readme
```
Decode base64 content. Extract all arXiv IDs (pattern: `\d{4}\.\d{4,5}`).

For each arXiv ID found, fetch the paper page to extract all co-authors and their affiliations:
```
WebSearch: "arxiv {arXiv_id} authors affiliations"
```
or fetch `https://arxiv.org/abs/{arXiv_id}` directly.

**Merge commit authors + paper authors** into a single candidate list. Mark each person's source:
- `[commit]` — appeared in git history
- `[paper]` — listed as paper author but no commit
- `[both]` — commit author AND paper author

The paper author list often reveals the full team, including leads who don't push code directly
but are responsible for data pipeline design and architecture.

---

### Step 2: Enrich each contributor profile

For each unique login (commit authors) or name (paper-only authors):

**For GitHub logins:**
```
GET /users/{login}
```
Extract: `name`, `email`, `company`, `location`, `bio`, `blog`, `html_url`, `followers`.

**For paper-only authors (no GitHub login):**
Search `"{name} {affiliation} github"` to find their GitHub profile.

---

### Step 3: Email extraction — 4-source strategy

Attempt each source in order, stop when found:

| Source | Method | Example yield |
|---|---|---|
| **① commit metadata** | From Step 1B commit data: `commit.author.email` | `renzhongwei@bytedance.com` |
| **② paper corresponding author** | Fetch arXiv PDF/abstract, look for footnote `*Corresponding author` or `†` with email | `jinxiaojie@bytedance.com` |
| **③ GitHub profile** | `user.email` field | `user@company.com` |
| **④ personal homepage** | Fetch `user.blog` URL, scan HTML for `mailto:` hrefs and `@` patterns | `name@lab.edu` |

Discard emails matching `*@users.noreply.github.com`.
If only domain is found (e.g. from Google Scholar "Verified email at bytedance.com"),
record as `? @ bytedance.com` — still useful as a partial lead.

Count how many PRs each login contributed (from Step 1 data).

### Step 4: Identify Chinese contributors

Score each contributor against these signals. You're looking for people who are ethnically Chinese,
including mainland China, Taiwan, Hong Kong, Singapore, and overseas Chinese diaspora.

**High confidence 🟢** — mark if ANY of these match:
- `name` contains Chinese characters (Unicode \u4e00–\u9fff)
- `name` starts with a common Chinese surname (Zhang, Wang, Li, Liu, Chen, Yang, Huang, Zhao, Wu,
  Zhou, Xu, Sun, Ma, Zhu, Lin, He, Guo, Luo, Zheng, Xie, Liang, Song, Tang, Cao, Deng, Han, Feng,
  Dong, Jiang, Peng, Xiao, Cheng, Wei, Cai, Lu, Kong, Mao, Shao, Wan, Yin, Shi, Yu, Qian, Yao,
  Fang, Hu, Bai, Cui, Zhu, Qin, Shen, Ding)

**Medium confidence 🟡** — mark if ANY of these match (but none of the high signals):
- `login` contains a Chinese surname from the list above (case-insensitive, e.g. `liulehui`,
  `zhangsan`, `XuQianJin-Stars`)
- `location` mentions: China, Beijing, Shanghai, Shenzhen, Hangzhou, Chengdu, Guangzhou, Wuhan,
  Nanjing, Chongqing, Suzhou, Xi'an, Tianjin, Hong Kong, Taiwan, Singapore, 北京, 上海, 深圳
- `company` mentions a Chinese tech company: ByteDance, Baidu, Alibaba, Tencent, Huawei, Meituan,
  JD.com, Didi, Kuaishou, Ant Group, NetEase, PingCAP, 字节, 百度, 阿里, 腾讯, 华为, 美团
- bio contains Chinese text or mentions Chinese institutions

**Skip** contributors with no signals. Include all high + medium confidence contributors in output.

---

### Step 5: Build output

For each Chinese contributor, generate their LinkedIn search URL:
```
https://www.linkedin.com/search/results/people/?keywords={URL-encoded name or login}
```
Prefer real name over login for better search results.

---

### Step 6: Write the output file

Save to: `github-digger-{repo-name}-{YYYY-MM-DD}.md` in the current working directory.

Use exactly this structure:

---

```markdown
# {owner}/{repo} — 华人贡献者挖角名单

## 找人结果概览

**扫描策略：** {PR模式 / Commit模式（企业内部 research repo，PR数 < 10）}  
**arXiv 论文溯源：** {是/否} — {论文标题，如有}

从 {repo} 项目中，找到 **{high} 位高置信度华人贡献者**，以及额外 **{medium} 位待确认的候选**。

---

## 第一梯队：高置信度华人贡献者 ({high}位)

| 排名 | GitHub | 真实姓名 | 邮箱 | 邮箱来源 | 公司 | 地点 | 来源 | 贡献 | 最新活跃 |
|------|--------|---------|------|---------|------|------|------|------|---------|
| 1 | [login](github_url) | name | email or ⚠️ | commit/paper/profile/homepage | company | location | [commit]/[paper]/[both] | n commits or 论文作者 | YYYY-MM-DD |

---

## 第二梯队：中置信度候选 ({medium}位)

| GitHub | 真实姓名 | 邮箱 | 公司 | 地点 | 来源 | 可能身份 |
|--------|---------|------|------|------|------|---------|
| [login](github_url) | name or — | — | — | — | [commit]/[paper] | 拼音特征/location信号 |

---

## 挖角策略建议

### 优先联系（第一梯队 - 高置信度 + 有邮箱）

**1. {name}** (@{login})
- 📧 {email}（邮箱来源：{commit metadata / 论文对应作者脚注 / GitHub profile / 个人主页}）
- 🏢 {company} · {location}
- 💻 贡献：{n commits / 论文作者} — 最近活跃：{latest_date}
- 🔗 GitHub: {github_url} | LinkedIn: {linkedin_search_url}

---

### 联系方式

1. **有邮箱的候选人**：直接发邮件，展示对其具体贡献的了解（commit message / 论文模块）
2. **无邮箱的候选人**：
   - 访问 GitHub profile，查看个人网站或联系方式
   - 在 LinkedIn 搜索，用真实姓名或用户名
   - 查看其开源项目的 discussions / issues 留言
3. **仅在论文中出现（无 GitHub）**：通过 arXiv 脚注 email 或机构主页联系

---

### 推荐的挖角话术

针对 **{role}** 的邮件/LinkedIn 消息（英文）：

> Hi {name},
>
> I noticed your contributions to {repo} — particularly your work on [具体commit描述 or 论文模块].
> We're building {role} at [Your Company], and your expertise in [相关技术领域] is exactly what
> we're looking for.
>
> We're [公司背景] focused on [核心方向]. The role offers [独特优势].
>
> Would you be open to a quick 15-minute chat?

中文版本：

> 你好 {name}，
>
> 我注意到你在 {repo} 的贡献，特别是 [具体commit/论文模块]。我们在 [公司名] 正在建设 {role}，
> 你在 [技术领域] 的经验正好是我们需要的。
>
> 我们是一家 [融资阶段] 的公司，专注于 [方向]。[独特卖点]。
>
> 有兴趣聊一下吗？

---

## 进一步扩大搜索范围

如果需要更多候选，可考虑：

1. **同一企业 Org 的其他 repo** — `org:{org_name}` 下同类项目，共享工程师池
2. **论文共同作者** — 追溯论文其他作者的 GitHub / 个人主页
3. **arXiv 同领域论文** — 搜索同方向近期论文，追踪第一作者/对应作者
4. **相关开源生态** — 引用该 repo 的其他项目贡献者

---

## 数据来源与说明

- **扫描时间：** {YYYY-MM-DD}
- **扫描策略：** {PR模式（最近 N 个 closed PR）/ Commit模式（最近 N 个 commits + arXiv 溯源）}
- **发现总数：** {total} 位候选人（高 {high} / 中 {medium}）
- **邮箱覆盖率：** {email_count}/{total} ({email_rate}%)
- **邮箱来源分布：** commit metadata {n} / 论文脚注 {n} / GitHub profile {n} / 个人主页 {n}

---

*Generated by github-digger skill*
```

---

## After writing the file

Tell the user:
1. Where the file was saved
2. Scan strategy used (PR mode vs Commit mode) and why
3. Total candidates found and breakdown (high/medium confidence)
4. How many have confirmed emails and which source each came from
5. If arXiv tracing was triggered: how many additional paper-only authors were found beyond commit authors
6. If fewer than 3 Chinese contributors were found, suggest: scanning more commits, trying related
   repos in the same org, or checking the arXiv paper's full author list

If the user asks for a personalized message for a specific candidate, generate one using their
actual commit messages or paper module names, company, and background. Keep it under 150 words.
