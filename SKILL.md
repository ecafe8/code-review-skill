---
name: code-review-excellence
description: |
  Provides comprehensive code review guidance for React 19, Vue 3, Rust, TypeScript, Java, Python, and C/C++.
  Helps catch bugs, improve code quality, and give constructive feedback.
  Use when: reviewing pull requests, conducting PR reviews, code review, reviewing code changes,
  establishing review standards, mentoring developers, architecture reviews, security audits,
  checking code quality, finding bugs, giving feedback on code.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash      # 运行 lint/test/build 命令验证代码质量
  - WebFetch  # 查阅最新文档和最佳实践
---

# Code Review Excellence

Transform code reviews from gatekeeping to knowledge sharing through constructive feedback, systematic analysis, and collaborative improvement.

## When to Use This Skill

- Reviewing pull requests and code changes
- Establishing code review standards for teams
- Mentoring junior developers through reviews
- Conducting architecture reviews
- Creating review checklists and guidelines
- Improving team collaboration
- Reducing code review cycle time
- Maintaining code quality standards

## Core Principles

### 1. The Review Mindset

**Goals of Code Review:**
- Catch bugs and edge cases
- Ensure code maintainability
- Share knowledge across team
- Enforce coding standards
- Improve design and architecture
- Build team culture

**Not the Goals:**
- Show off knowledge
- Nitpick formatting (use linters)
- Block progress unnecessarily
- Rewrite to your preference

### 2. Effective Feedback

**Good Feedback is:**
- Specific and actionable
- Educational, not judgmental
- Focused on the code, not the person
- Balanced (praise good work too)
- Prioritized (critical vs nice-to-have)

```markdown
❌ Bad: "This is wrong."
✅ Good: "This could cause a race condition when multiple users
         access simultaneously. Consider using a mutex here."

❌ Bad: "Why didn't you use X pattern?"
✅ Good: "Have you considered the Repository pattern? It would
         make this easier to test. Here's an example: [link]"

❌ Bad: "Rename this variable."
✅ Good: "[nit] Consider `userCount` instead of `uc` for
         clarity. Not blocking if you prefer to keep it."
```

### 3. Review Scope

**What to Review:**
- Logic correctness and edge cases
- Security vulnerabilities
- Performance implications
- Test coverage and quality
- Error handling
- Documentation and comments
- API design and naming
- Architectural fit

**What Not to Review Manually:**
- Code formatting (use Prettier, Black, etc.)
- Import organization
- Linting violations
- Simple typos

## Review Process

### Phase 1: Context Gathering (2-3 minutes)

Before diving into code, understand:
1. Read PR description and linked issue
2. Check PR size (>400 lines? Ask to split)
3. Review CI/CD status (tests passing?)
4. Understand the business requirement
5. Note any relevant architectural decisions

### Phase 2: High-Level Review (5-10 minutes)

1. **Architecture & Design** - Does the solution fit the problem?
   - For significant changes, consult [Architecture Review Guide](reference/architecture-review-guide.md)
   - Check: SOLID principles, coupling/cohesion, anti-patterns
2. **Performance Assessment** - Are there performance concerns?
   - For performance-critical code, consult [Performance Review Guide](reference/performance-review-guide.md)
   - Check: Algorithm complexity, N+1 queries, memory usage
3. **File Organization** - Are new files in the right places?
4. **Testing Strategy** - Are there tests covering edge cases?

### Phase 3: Line-by-Line Review (10-20 minutes)

For each file, check:
- **Logic & Correctness** - Edge cases, off-by-one, null checks, race conditions
- **Security** - Input validation, injection risks, XSS, sensitive data
- **Performance** - N+1 queries, unnecessary loops, memory leaks
- **Maintainability** - Clear names, single responsibility, comments

### Phase 4: Summary & Decision (2-3 minutes)

1. Summarize key concerns
2. Highlight what you liked
3. Make clear decision:
   - ✅ Approve
   - 💬 Comment (minor suggestions)
   - 🔄 Request Changes (must address)
4. Offer to pair if complex

### Phase 5: Generate Review Report (2-3 minutes)

导出本次 review 结果为结构化 Markdown 文件，便于归档、搜索与后续跟进。

#### 输出文件规范

**文件命名**
```
review-YYYYMMDDTHHMMSS-<branch>-<short-sha>.md
```
- `YYYYMMDDTHHMMSS`: ISO8601 时间戳
- `<branch>`: 分支名（斜杠替换为下划线），例如 `feature_x`、`bugfix_auth`
- `<short-sha>`: 提交 hash 前 7 位，例如 `1a2b3c4`

示例：`review-20260317T143200-feature_x-1a2b3c4.md`

**文件编码与格式**
- UTF-8 编码
- CommonMark 格式
- 代码片段使用三重反引号并标注语言

**YAML Frontmatter（必须）**

```yaml
---
title: "简短标题描述本次审查"
repo: "仓库名"
branch: "feature/x/new-router"
commit: "1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p7q8r9s0t"
commit_short: "1a2b3c4"
author: "提交者 username"
reviewers: ["reviewer1", "reviewer2"]
date: "2026-03-17T14:32:00Z"
decision: "Request Changes"
ci_status: "pass"
severity_summary:
  blocking: 1
  important: 2
  nit: 3
tags: ["performance", "security", "architecture"]
---
```

**文档主体结构（固定顺序）**

1. **TL;DR** — 一句结论
   ```
   [决策符号] [主要问题] 【分支】 [提交HASH]
   示例：🔴 [blocking] 路由缓存竞态 【feature_x】 1a2b3c4
   ```

2. **概要（Summary）** — 变更目的与范围（1-3 段）
   ```
   - 涵盖的文件数、行数
   - 主要的功能变更或修复
   - 相关的业务需求或技术债务
   ```

3. **优点（What I liked）** — 列点，使用 🎉 标记
   ```
   - 🎉 设计简洁，易于测试
   - 🎉 充分的错误处理
   ```

4. **重要问题（Major Issues / Blocking）** — 🔴 标记，包含：
   - 问题标题
   - 影响范围与复现步骤
   - 文件定位：[文件路径](相对链接) 用于 vscode 快速跳转
   - 代码定位：[文件路径](相对链接) @ commit hash
   - 建议修复方案
   ```
   ## 🔴 [blocking] 路由缓存竞态
   
   **文件**：[src/router.ts](src/router.ts)
   **位置**：[src/router.ts](src/router.ts#L120-L150) @ 1a2b3c4
   
   **问题**：高并发下可能导致重复注册路由，引发路由冲突。
   
   **复现**：
   - 运行 `scripts/load-test.sh`
   - 观察日志中的 duplicate route 警告
   
   **建议修复**：
   - 使用 ReadWrite 锁保护缓存更新
   - 或改为原子操作（CAS）+重试机制
   ```

5. **次要问题（Important / Non-blocking）** — 🟡 标记
   ```
   ## 🟡 [important] 缺少单元测试
   
   **文件**：[src/middleware.ts](src/middleware.ts)
   **位置**：[src/middleware.ts](src/middleware.ts#L45-L60) @ 1a2b3c4
   
   **问题**：新增中间件没有对应的单元测试。
   
   **建议**：补充边界情况测试，至少覆盖：timeout、error response、header validation。
   ```

6. **小建议（Nit / Style）** — 🟢 标记
   ```
   ## 🟢 [nit] 变量命名建议
   
   **文件**：[src/utils.ts](src/utils.ts)
   **位置**：[src/utils.ts#L10](src/utils.ts#L10) @ 1a2b3c4：`uc` → `userCount` 提高可读性
   ```

7. **测试与验证（Tests & How to Verify）**
   ```
   ## 测试状态
   
   - ✅ 单元测试：通过 (45 cases)
   - ✅ 集成测试：通过
   - ❌ E2E 测试：1 个失败（超时）
   
   ## 本地验证步骤
   
   \`\`\`bash
   git checkout -b review/feature_x 1a2b3c4
   bun install
   bun run test
   bun run test:e2e
   \`\`\`
   ```

#### 存放位置与权限

- **位置**：存放到 `review-archive/` 子目录，避免污染主分支代码

## Review Techniques

### Technique 1: The Checklist Method

Use checklists for consistent reviews. See [Security Review Guide](reference/security-review-guide.md) for comprehensive security checklist.

### Technique 2: The Question Approach

Instead of stating problems, ask questions:

```markdown
❌ "This will fail if the list is empty."
✅ "What happens if `items` is an empty array?"

❌ "You need error handling here."
✅ "How should this behave if the API call fails?"
```

### Technique 3: Suggest, Don't Command

Use collaborative language:

```markdown
❌ "You must change this to use async/await"
✅ "Suggestion: async/await might make this more readable. What do you think?"

❌ "Extract this into a function"
✅ "This logic appears in 3 places. Would it make sense to extract it?"
```

### Technique 4: Differentiate Severity

Use labels to indicate priority:

- 🔴 `[blocking]` - Must fix before merge
- 🟡 `[important]` - Should fix, discuss if disagree
- 🟢 `[nit]` - Nice to have, not blocking
- 💡 `[suggestion]` - Alternative approach to consider
- 📚 `[learning]` - Educational comment, no action needed
- 🎉 `[praise]` - Good work, keep it up!

## Language-Specific Guides

根据审查的代码语言、框架，查阅对应的详细指南：

| Language/Framework | Reference File | Key Topics |
|-------------------|----------------|------------|
| **React** | [React Guide](reference/react.md) | Hooks, useEffect, React 19 Actions, RSC, Suspense, TanStack Query v5 |
| **TypeScript** | [TypeScript Guide](reference/typescript.md) | 类型安全, async/await, 不可变性 |
| **Hono** | [Hono Guide](reference/hono.md) | 中间件设计, 路由优化, 性能审查, 类型安全 RPC, 认证与安全, 应用结构 |
| **Drizzle ORM** | [Drizzle Guide](reference/drizzle.md) | Schema 定义, 查询构造器, 关系查询, Mutations 安全性, 事务完整性, drizzle-kit 迁移管理 |
| **CSS/Less/Sass** | [CSS Guide](reference/css-less-sass.md) | 变量规范, !important, 性能优化, 响应式, 兼容性 |
| **Python** | [Python Guide](reference/python.md) | 可变默认参数, 异常处理, 类属性 |
| **Java** | [Java Guide](reference/java.md) | Java 17/21 新特性, Spring Boot 3, 虚拟线程, Stream/Optional |
| **Go** | [Go Guide](reference/go.md) | 错误处理, goroutine/channel, context, 接口设计 |
| **C** | [C Guide](reference/c.md) | 指针/缓冲区, 内存安全, UB, 错误处理 |
| **C++** | [C++ Guide](reference/cpp.md) | RAII, 生命周期, Rule of 0/3/5, 异常安全 |
| **Vue 3** | [Vue Guide](reference/vue.md) | Composition API, 响应性系统, Props/Emits, Watchers, Composables |
| **Rust** | [Rust Guide](reference/rust.md) | 所有权/借用, Unsafe 审查, 异步代码, 错误处理 |
| **Qt** | [Qt Guide](reference/qt.md) | 对象模型, 信号/槽, 内存管理, 线程安全, 性能 |

## Additional Resources

- [Architecture Review Guide](reference/architecture-review-guide.md) - 架构设计审查指南（SOLID、反模式、耦合度）
- [Performance Review Guide](reference/performance-review-guide.md) - 性能审查指南（Web Vitals、N+1、复杂度）
- [Common Bugs Checklist](reference/common-bugs-checklist.md) - 按语言分类的常见错误清单
- [Security Review Guide](reference/security-review-guide.md) - 安全审查指南
- [Code Review Best Practices](reference/code-review-best-practices.md) - 代码审查最佳实践
- [PR Review Template](assets/pr-review-template.md) - PR 审查评论模板
- [Review Checklist](assets/review-checklist.md) - 快速参考清单
