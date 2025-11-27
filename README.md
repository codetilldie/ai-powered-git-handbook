# 完全由 AI 撰写的 Git 手册

一份完全由 Gemini AI 撰写、基于 Gemini CLI 生成和迭代、内容全面的 Git 学习指南，专为各级开发者设计。系统性地生成和筛选，旨在让开发者更直观、高效地掌握 Git。

## 简介

本手册旨在利用 Gemini AI 的力量，为开发者提供一个全面、易懂且持续更新的 Git 学习资源。无论您是初学者还是经验丰富的开发者，都能在这里找到所需的内容，系统地掌握 Git 的核心知识和高级技巧，从而在实际项目中更加得心应手。

本手册将涵盖 Git 的基础知识、高级功能、团队协作工作流、常见问题和最佳实践，旨在帮助开发者快速掌握 Git，并在实际项目中灵活运用。我们将采用简洁明了的语言，结合丰富的实例和代码示例，帮助读者更好地理解和掌握 Git。

我们致力于提供最全面和实用的 Git 学习资源，因此会不断更新和优化手册内容，以适应 Git 的最新发展和市场需求。

## 目录

- **[第一章：Git 入门](./handbook/01-getting-started.md)**
  - 1.1 什么是 Git？
  - 1.2 Git 的安装与配置
  - 1.3 开启你的 Git 之旅：`init` 与 `clone`
  - 1.4 核心工作流：三大区域
  - 1.5 基础命令：`status`, `add`, `commit`, `log`, `diff`

- **[第二章：Git 核心概念](./handbook/02-core-concepts.md)**
  - 2.1 理解分支 (Branch)
  - 2.2 合并分支：merge 与 rebase
  - 2.3 远程仓库：remote, push & pull
  - 2.4 使用标签 (Tag)

- **[第三章：Git 高级功能](./handbook/03-advanced-features.md)**
  - 3.1 储藏 (Stash)
  - 3.2 交互式变基 (Interactive Rebase)
  - 3.3 子模块 (Submodules) 与子树 (Subtree)
  - 3.4 Git Hooks
  - 3.5 cherry-pick, reflog 等高级命令

- **[第四章：团队协作与最佳实践](./handbook/04-collaboration-and-best-practices.md)**
  - 4.1 流行的 Git 工作流
  - 4.2 编写清晰的 Commit Message
  - 4.3 Code Review 最佳实践 (Pull Request)
  - 4.4 如何处理合并冲突

- **[第五章：常见问题与解决方案 (FAQ)](./handbook/05-faq.md)**
  - 5.1 如何撤销一次错误的提交？
  - 5.2 `.gitignore` 文件为什么不起作用？
  - 5.3 如何修改历史提交信息？

- **[第六章：深入底层 - 从数据库视角理解 Git](./handbook/06-git-as-a-database.md)**
  - 6.1 `objects` (对象表) - Git 的数据核心
  - 6.2 `refs` (引用表) - 人类可读的指针
  - 6.3 `HEAD` (头指针表) - “你在这里”的指示牌
  - 6.4 `index` (索引/暂存区表) - 下一次提交的“购物篮”
  - 6.5 复杂操作的数据库视角

- **[第七章：附录 - 资源与工具](./handbook/07-appendix-resources.md)**
  - 官方与权威文档
  - 在线练习平台
  - GUI 客户端工具推荐
  - 社区与支持
