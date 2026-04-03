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

- **repo** (required): GitHub repo in `owner/repo` format, or a full GitHub URL
- **limit** (optional): How many recent closed PRs to scan. Default: 100
- **role** (optional): The role you're recruiting for. Default: "Data Infra Tech Lead"

If the user provides a full URL like `https://github.com/ray-project/ray`, extract `ray-project/ray`.
If only a repo name is given (e.g., "vllm"), ask the user to confirm the owner before proceeding.

## Execution

Work through these steps sequentially. Use `WebFetch` for all GitHub API calls.

### Step 1: Fetch closed PRs

```
GET https://api.github.com/repos/{owner}/{repo}/pulls?state=closed&sort=updated&direction=desc&per_page=100
```

Collect unique contributor `login` values from `pull.user.login`. Skip bots (logins containing
`[bot]` or ending in `-bot`). If `limit` is less than 100, fetch only the first page accordingly.

**Rate limit handling**: If you get a 403 or see `X-RateLimit-Remaining: 0`, stop immediately and
tell the user:
> GitHub API rate limit reached (60 req/hour without authentication). Add a token at
> https://github.com/settings/tokens and pass it to me as "use token: ghp_xxx" to continue.

### Step 2: Enrich each contributor profile

For each unique login:

```
GET https://api.github.com/users/{login}
```

Extract: `name`, `email`, `company`, `location`, `bio`, `blog` (personal website), `html_url`,
`avatar_url`.

**Email fallback** — only if `email` is null:
```
GET https://api.github.com/repos/{owner}/{repo}/commits?author={login}&per_page=3
```
Take `items[0].commit.author.email`. Discard if it matches `*@users.noreply.github.com`.

Count how many PRs each login contributed (from Step 1 data).

### Step 3: Identify Chinese contributors

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

**Skip** contributors with no signals. Include all high + medium confidence contributors in output.

### Step 4: Build output

For each Chinese contributor, generate their LinkedIn search URL:
```
https://www.linkedin.com/search/results/people/?keywords={URL-encoded name or login}
```
Prefer real name over login for better search results.

### Step 5: Write the output file

Save to: `github-digger-{repo-name}-{YYYY-MM-DD}.md` in the current working directory.

Use exactly this structure:

---

```markdown
# {owner}/{repo} — 华人贡献者挖角名单

## 找人结果概览

从 {repo} 项目的最近 closed PR 中，找到 **{high} 位高置信度华人贡献者**，以及额外 **{medium} 位待确认的候选**。

---

## 第一梯队：高置信度华人贡献者 ({high}位)

| 排名 | GitHub | 真实姓名 | 邮箱 | 公司 | 地点 | 网站 | 置信度 | PR数 | 最新活跃 |
|------|--------|---------|------|------|------|------|--------|------|---------|
| 1 | [login](github_url) | name or — | email or ⚠️ | company or — | location or — | [网站](blog_url) or — | 🟢高 | n | YYYY-MM-DD |

---

## 第二梯队：中置信度候选 ({medium}位)

| GitHub | 真实姓名 | 邮箱 | 公司 | 地点 | 置信度 | PR数 | 可能身份 |
|--------|---------|------|------|------|--------|------|---------|
| [login](github_url) | name or — | — | — | — | 🟡中 | n | 拼音特征/location信号 |

---

## 挖角策略建议

### 优先联系（第一梯队 - 高置信度 + 有邮箱）

这些是最优先的目标：

**1. {name}** (@{login})
- 📧 {email} （有邮箱，可直接联系）
- 🏢 {company} · {location}
- 💻 PR 贡献：{pr_count} 个，最近活跃：{latest_pr_date}
- 🔗 GitHub Profile: {github_url}

**2. {name}** (@{login})
- 📧 ⚠️ 无公开邮箱 → 可通过 GitHub profile 联系或 LinkedIn 搜索
- 🏢 {company} · {location}
- 💻 PR 贡献：{pr_count} 个，最近活跃：{latest_pr_date}
- 🔗 GitHub Profile: {github_url} | LinkedIn: {linkedin_search_url}

---

### 联系方式

1. **有邮箱的候选人**：直接发邮件，展示对其 GitHub 贡献的了解
2. **无邮箱的候选人**：
   - 访问 GitHub profile，查看个人网站或联系方式
   - 在 LinkedIn 搜索，用真实姓名或用户名
   - 查看其开源项目的 discussions / issues 留言
3. **公司信号**：通过公司内部邮件或 LinkedIn 找到对应团队

---

### 推荐的挖角话术

针对 **{role}** 的 LinkedIn 消息（英文）：

> Hi {name},
>
> I noticed your contributions to {repo}, particularly your work on [具体PR名]. We're building
> {role} at [Your Company], and your expertise in [相关技术领域] is exactly what we're looking for.
>
> We're a [公司背景：Pre-IPO/Series X/成长阶段] focused on [核心方向]. The role offers [独特优势：
> 直接汇报、高度自主权、技术深度等].
>
> Would you be open to a quick 15-minute chat to learn more about what we're building?
>
> Looking forward to hearing from you!

中文版本（如果候选人是中国开发者）：

> 你好 {name}，
>
> 我注意到你在 {repo} 上的贡献，特别是 [具体PR名]。我们在 [公司名] 正在建设 {role}，
> 你在 [技术领域] 的经验正好是我们需要的。
>
> 我们是一家 [融资阶段] 的公司，专注于 [方向]。这个职位 [独特卖点：直接汇报创始人、充分自主权等]。
> 薪酬范围：[具体数字]
>
> 有兴趣聊一下吗？随时可以。

---

## 进一步扩大搜索范围

如果需要更多候选，可考虑：

1. **其他头部开源项目**
   - vLLM、LLaMA、Hugging Face transformers
   - Databricks、MLflow 相关项目
   - PyTorch、TensorFlow 子项目

2. **特定技术领域**
   - 分布式系统：etcd、Kubernetes、TiDB
   - AI/ML：fairseq、Megatron-LM、ColossalAI
   - Data：Apache Spark、Parquet、Arrow

3. **学术和会议渠道**
   - arXiv 论文作者（按领域搜索）
   - MLSys、NeurIPS、ICML 讲者

4. **公司官方团队**
   - 项目官方治理页面（如 Ray docs/governance）
   - 核心贡献者通常在公司员工名单中

---

## 数据来源与说明

- **扫描时间：** {YYYY-MM-DD}
- **扫描范围：** 最近 {limit} 个 closed PR
- **发现总数：** {total} 位候选人（高 {high} / 中 {medium}）
- **邮箱覆盖率：** {email_count}/{total} ({email_rate}%)

### 关于邮箱

所有邮箱均来自 GitHub 公开数据（profile + commit 元数据），不涉及隐私信息。
约 20-30% 的 GitHub 用户会公开邮箱，其余候选人可通过其他渠道（LinkedIn、个人网站等）联系。

---

*Generated by github-digger skill*
```

---

## After writing the file

Tell the user:
1. Where the file was saved
2. Total candidates found and breakdown (high/medium confidence)
3. How many have public emails
4. If fewer than 3 Chinese contributors were found, suggest: scanning more PRs (increase limit),
   trying a related repo, or that this project may have few Chinese contributors

If the user asks for a personalized message for a specific candidate, generate one using their
actual PR titles, company, and background. Keep it under 150 words and conversational, not salesy.
