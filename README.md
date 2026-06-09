# 🧭 CLI Design Guide for LLMs & Agents

> 面向 LLM 与 AI Agent 的 CLI 设计指南 & 行为规范

[![License](https://img.shields.io/badge/license-CC_BY--SA_4.0-blue.svg)](LICENSE)

**可发现 · 可预测 · 可恢复 · 可解析 · 可契约化**  
*Discoverable · Predictable · Recoverable · Parsable · Contract-driven*

---

## 这是什么？ / What is this?

一套帮助开发者构建**同时对人类和 AI Agent 友好的命令行工具**的设计指南与正式规范。涵盖信息架构、输出契约、错误模型、非交互执行、结构化输出、批量任务语义、安全边界、配置管理等关键主题。

A design guide and formal specification for building command-line tools that are **friendly to both humans and AI agents**. Covers information architecture, output contracts, error models, non-interactive execution, structured output, batch task semantics, safety boundaries, configuration management, and more.

---

## 目录结构 / Structure

```
├── README.md
├── CN/
│   ├── llm-friendly-cli-design-guide-public.md   ← 设计指南（中文）
│   └── llm-friendly-cli-spec.md                  ← 行为规范（中文）
└── EN/
    ├── llm-friendly-cli-design-guide-public.md   ← Design Guide (English)
    └── llm-friendly-cli-spec.md                  ← Specification (English)
```

| 文档 / Document | 中文 (CN) | English (EN) |
|---|---|---|
| 设计指南 / Design Guide | [CN/llm-friendly-cli-design-guide-public.md](CN/llm-friendly-cli-design-guide-public.md) | [EN/llm-friendly-cli-design-guide-public.md](EN/llm-friendly-cli-design-guide-public.md) |
| 行为规范 / Specification | [CN/llm-friendly-cli-spec.md](CN/llm-friendly-cli-spec.md) | [EN/llm-friendly-cli-spec.md](EN/llm-friendly-cli-spec.md) |

---

## 两份文档的关系 / How the Two Documents Relate

- **设计指南 (Design Guide)**：以实践视角展开，适合通读，包含大量示例和反模式说明。分为基础要求、推荐实践、进阶能力三个层级。
- **行为规范 (Specification)**：以规范性术语 (MUST / SHOULD / MAY) 编写，适合作为团队评审 checklist 或 AI 代码生成的约束输入。

> - **Design Guide**: Practice-oriented, suitable for reading through. Contains extensive examples and anti-pattern illustrations. Organized into three tiers: baseline requirements, recommended practices, and advanced capabilities.
> - **Specification**: Written in normative terminology (MUST / SHOULD / MAY). Suitable as a team review checklist or as constraint input for AI code generation.

---

## 适用场景 / Use Cases

- 🆕 从零设计一个新的 CLI 工具
- 🔧 改造现有 CLI 使其对 agent 更友好
- 🤖 作为 AI 代码生成的系统提示词/规范输入
- ✅ 团队 CLI 代码评审的验收标准
- 📋 MCP Server / Plugin / IDE Agent 的接口设计参考

---

## 许可证 / License

CC BY-SA 4.0 — 欢迎分享与改编，需署名并以相同方式共享。

---

## 关于 / About

本指南由 [Marks](https://github.com/Marksooxx) 编写。如果你觉得有用，欢迎 ⭐ Star 或分享给需要的团队。

*Written by [Marks](https://github.com/Marksooxx). If you find this useful, consider giving it a ⭐ or sharing it with teams that might benefit.*
