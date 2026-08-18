# Contributing to System Design Interview Guide

# 贡献指南

Thank you for your interest in contributing! This guide is community-driven and we welcome all forms of contribution.

感谢你有兴趣贡献！本指南由社区驱动，我们欢迎所有形式的贡献。

## Ways to Contribute / 贡献方式

### 1. Report Issues / 报告问题

Found a mistake? Missing content? Open an [issue](https://github.com/liangzhengtao/system-design-interview/issues).

发现问题？缺少内容？请提交 [issue](https://github.com/liangzhengtao/system-design-interview/issues)。

### 2. Improve Content / 改进内容

- Fix typos or grammatical errors
- Add ASCII diagrams
- Improve explanations
- Add capacity estimation numbers
- Translate content

### 3. Add New Designs / 添加新设计

Follow the existing structure:

```
designs/your-design.md
```

Use the **RESHADED** framework:

1. **R**equirements — Functional & non-functional
2. **E**stimation — Back-of-envelope calculations
3. **S**torage design — Database schema & data model
4. **H**igh-level design — Architecture diagram
5. **A**PI design — RESTful endpoints
6. **D**etailed design — Component deep dive
7. **E**valuation — Trade-offs & alternatives
8. **D**istinctive component — What makes this system unique

### 4. Bilingual Content / 双语内容

Every file must have both English and 中文 sections. Place the 中文 version at the end of the file under a `## 中文版本` heading.

## Getting Started / 开始贡献

```bash
# 1. Fork the repository
# 2. Clone your fork
git clone https://github.com/<your-username>/system-design-interview.git

# 3. Create a branch
git checkout -b feature/your-feature

# 4. Make your changes
# 5. Commit with a clear message
git commit -m "docs: add rate limiter design"

# 6. Push and create a Pull Request
git push origin feature/your-feature
```

## Style Guide / 风格指南

### Markdown

- Use `#` for the title (one per file)
- Use `##` for major sections
- Use `###` for subsections
- Keep lines under 120 characters where possible

### ASCII Diagrams

- Use box-drawing characters: `┌`, `┐`, `└`, `┘`, `─`, `│`, `├`, `┤`, `┬`, `┴`, `┼`
- Align components properly
- Label all arrows and connections
- Keep diagrams readable at 80-character width

Example:

```
┌──────────┐      ┌──────────┐      ┌──────────┐
│  Client  │─────▶│  Server  │─────▶│ Database │
└──────────┘      └──────────┘      └──────────┘
```

### Capacity Estimation

Always include:

- Daily active users (DAU)
- Read/write ratio
- Storage per record
- Bandwidth requirements
- QPS estimates

### Trade-off Analysis

Format as:

| Approach | Pros | Cons |
|----------|------|------|
| Option A | ...  | ...  |
| Option B | ...  | ...  |

## Pull Request Process / PR 流程

1. Ensure your changes follow the style guide
2. Update the README.md if adding new designs
3. Include both English and 中文 content
4. Add ASCII diagrams for any architecture
5. Request review from maintainers

## Code of Conduct / 行为准则

Please follow our [Code of Conduct](CODE_OF_CONDUCT.md).

请遵守我们的 [行为准则](CODE_OF_CONDUCT.md)。

## Questions? / 有问题？

Open an issue or reach out to the maintainers.

提交 issue 或联系维护者。

---

Thank you for making this guide better! 🙏

感谢你让这个指南变得更好！🙏
