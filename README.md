# GitHub Digger

> Find Chinese contributors from GitHub open source projects for talent recruiting and sourcing.

## Overview

GitHub Digger is a Claude skill that automatically scans GitHub repositories for Chinese contributors and generates professional, recruitment-ready candidate lists with contact information.

## Features

✨ **Automatic Chinese Contributor Identification**
- Rules-based detection (Chinese characters, pinyin surnames, location signals, company info)
- Multi-level confidence scoring (high/medium)

📧 **Contact Information Enrichment**
- Public emails from GitHub profiles and commit metadata
- Personal websites (blog links)
- LinkedIn search links

📋 **Professional Candidate Lists**
- Tiered by confidence level
- Detailed contribution history
- Ready-to-use recruiting copy

💬 **Recruitment Copy Templates**
- English LinkedIn messages
- Chinese email templates
- Personalized by candidate background

## Usage

```
帮我从 vllm-project/vllm 找华人贡献者
```

or

```
从 ray-project/ray 找华人工程师，我们招 Data Infra Tech Lead
```

## Output Format

The skill generates a markdown report with:

1. **Result Overview** — Quick summary of findings
2. **Tier 1 Table** — High confidence candidates with full details
3. **Tier 2 Table** — Medium confidence candidates  
4. **Recruitment Strategy** — Priority contacts, communication channels, message templates
5. **Search Expansion Suggestions** — Recommended projects and channels
6. **Data Source & Privacy** — Email coverage rate and data transparency

### Example Output

```markdown
# vllm-project/vllm — 华人贡献者挖角名单

## 找人结果概览

从 vllm-project/vllm 项目的最近 closed PR 中，找到 **10 位高置信度华人贡献者**...

## 第一梯队：高置信度华人贡献者 (10位)

| 排名 | GitHub | 真实姓名 | 邮箱 | 公司 | 地点 | 置信度 | PR数 |
|------|--------|---------|------|------|------|--------|------|
| 1 | [youkaichao](https://github.com/youkaichao) | Kaichao You | — | Inferact | SF | 🟢高 | 15+ |
...
```

## Technical Details

### Identification Method

Three-layer signal detection:

1. **Chinese Characters** — Names with CJK Unicode characters → high confidence
2. **Pinyin Surnames** — Common Chinese surname pinyin (xu, wang, li, zhang, etc.) → medium confidence  
3. **Location/Company Signals**
   - Location mentions: China, Beijing, Shanghai, HK, Taiwan, Singapore
   - Company mentions: ByteDance, Alibaba, Tencent, Huawei, etc.

### Data Sources

- GitHub API: `/repos/{owner}/{repo}/pulls` (closed PRs)
- GitHub User Profiles: `/users/{login}`
- GitHub Commits: `/repos/{owner}/{repo}/commits` (for email fallback)

### Email Coverage

Approximately 20-30% of GitHub users publicly share email addresses. Remaining candidates can be contacted via:
- LinkedIn profile/search
- Personal website/blog
- GitHub issues/discussions
- Company internal channels

### Rate Limiting

- **Without GitHub Token**: 60 requests/hour (GitHub's standard limit)
- **With GitHub Token**: 5,000 requests/hour
- Recommended: Configure token for larger scans

## Test Results

### vLLM Scan (2026-04-03)
- **Candidates Found**: 14 (10 high confidence / 4 medium)
- **Email Coverage**: 28.6% (4/14)
- **Top Candidates**: 
  - MengqingCao (Huawei/Ascend, email known, 28+ PRs)
  - youkaichao (Inferact CTO, 15+ PRs, vLLM core maintainer)

### Ray Scan (2026-04-03)
- **Candidates Found**: 14 (8 high confidence / 6 medium)
- **Notable**: Identified 8 official team members from Ray governance page
- **Most Active**: rueian (7 PRs)

## Parameters

When invoking the skill:

- **repo** (required): GitHub repo as `owner/repo` or full URL
- **limit** (optional): Number of recent closed PRs to scan. Default: 100
- **role** (optional): Position you're recruiting for. Default: "Data Infra Tech Lead"

Example:
```
帮我从 pytorch/pytorch 找华人贡献者，限制最近 150 个 PR，我们招机器学习工程师
```

## Use Cases

- **AI/ML Talent Sourcing** — Find engineers from Ray, vLLM, PyTorch, etc.
- **Distributed Systems Experts** — Scan projects like etcd, Kubernetes, TiDB
- **Data Infrastructure** — Identify contributors from Spark, Arrow, DuckDB
- **Geographic Expansion** — Target Chinese engineers from open source projects
- **Company-Specific Recruiting** — Find people who have worked on key tech

## Privacy & Ethics

✅ **All data is public**
- GitHub profiles and commit metadata are publicly available
- No private information is accessed or disclosed
- Email addresses are shared voluntarily by users on their GitHub profiles

⚠️ **Responsible Use**
- Use for legitimate recruiting purposes only
- Respect platform terms of service
- Personalize outreach (don't spam)
- Verify identities before contacting

## Limitations

- **Email Coverage**: ~20-30% of GitHub users have public emails
- **PR-Based Scope**: Scans recent closed PRs; may miss long-time contributors
- **Confidence Scoring**: Rules-based detection can have false positives/negatives
- **Language**: Currently optimized for Chinese name detection

### Recommended Supplements

To find additional candidates:
1. Check project's official governance/team page
2. Look at early commit history for founding team
3. Review academic papers (arXiv) in related domains
4. Scan conference speakers (NeurIPS, ICML, MLSys)
5. Use LinkedIn for complementary searches

## Future Improvements

- [ ] Add Qwen AI model for enhanced name classification
- [ ] Support scanning multiple repos and aggregating results
- [ ] Auto-import from official team/governance pages
- [ ] Export to CSV/Excel with custom templates
- [ ] LinkedIn API integration for direct profile lookup
- [ ] Timezone detection for better timing of outreach

## Development

### Structure

```
github-digger/
├── SKILL.md           # Claude skill definition
├── github-digger.skill # Packaged skill file
├── README.md          # This file
└── EXAMPLES.md        # Usage examples and sample outputs
```

### How It Works

The skill is self-contained in `SKILL.md` and uses only built-in Claude tools:
- **WebFetch** for GitHub API calls
- **Output** as markdown files

No external dependencies or backend required.

### Contributing

Found a bug or have a suggestion?
- File an issue
- Submit a PR with improvements
- Share feedback on accuracy or coverage

## License

MIT

## Contact

For questions or feedback about this skill:
- GitHub Issues: [github-digger/issues](https://github.com/yourusername/github-digger/issues)
- Email: [your-email]

---

**Made with ❤️ for AI recruiting**

Built as a Claude skill - leverage the power of Claude API for talent sourcing.
