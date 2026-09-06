# concept-workshop：用项目级 Skill 沉淀概念学习

用 WorkBuddy 的项目级 Skill「concept-learning-kit」把陌生概念变成结构化、可核查的学习资料。本仓库包含一个可复用的 Skill 和按它生成的四份学习材料。

## 仓库用途

1. **沉淀一个可复用的学习 Skill**：接收任意新概念，自动生成格式统一的概念学习资料（不针对特定主题）；
2. **产出三份概念学习资料**：Agent、大模型的上下文、Skill；
3. **说明三者的关系**：上下文如何支撑 Agent 决策、Skill 如何沉淀流程知识。

## 目录结构

```
concept-workshop/
├── .workbuddy/
│   └── skills/
│       └── concept-learning-kit/
│           └── SKILL.md          # 项目级 Skill（本仓库核心）
├── learning-materials/
│   ├── agent.html                # 概念一：Agent
│   ├── llm-context.html          # 概念二：大模型的上下文
│   ├── skill.html                # 概念三：Skill
│   └── （concept-relationship.md 在仓库根目录）
├── concept-relationship.md       # 三概念关系说明（含 Mermaid 图）
├── README.md
└── .gitignore
```

## Skill 存放路径与调用方法

- **路径**：`.workbuddy/skills/concept-learning-kit/SKILL.md`（项目级，随仓库共享）
- **元数据**：`name: concept-learning-kit`，`description: 概念学习资料生成器……`
- **在 WorkBuddy 中调用**：
  1. 在 WorkBuddy 中打开本仓库（作为工作目录）；
  2. 直接对话，例如：`使用项目级 Skill「concept-learning-kit」，学习概念"检索增强生成（RAG）"，深度"入门+机制"`；
  3. Agent 会匹配 Skill 的 description 并加载完整指令，按九章节模板生成资料到 `learning-materials/`。
- 也可以直接打开 SKILL.md 阅读其提示词设计（适用场景 / 输入 / 生成步骤 / 输出结构 / 资料来源要求 / 自检要求）。

## 已生成的学习资料

| 文件 | 主题 | 内容要点 |
|---|---|---|
| [learning-materials/agent.html](learning-materials/agent.html) | Agent | 个人解释、感知-规划-工具-观察循环、会议记录整理案例、Workflow/Chatbot 辨析、自测题 |
| [learning-materials/llm-context.html](learning-materials/llm-context.html) | 大模型的上下文 | 窗口组成与 token 计量、无状态与重放、Lost in the Middle、Agent 长任务案例 |
| [learning-materials/skill.html](learning-materials/skill.html) | Skill | SKILL.md 结构、渐进式披露、与 MCP/提示词辨析、concept-learning-kit 本身的案例 |
| [concept-relationship.md](concept-relationship.md) | 三者关系 | 角色对照表、两幅 Mermaid 图、上下文如何影响 Agent、Skill 如何沉淀知识 |

每份资料均包含：一句话个人解释、学习目标、核心问题、定义与边界、核心机制、应用案例、易混淆问题与使用边界、自测题、可核查的参考来源。

## 使用 AI 后，我做了哪些人工核查与修改

AI（WorkBuddy + concept-learning-kit Skill）生成了全部初稿，以下是我逐条人工核查与修改的记录：

1. **来源链接逐一验证**：用工具对全部 11 条引用链接做了可达性测试（HTTP 状态码）；
   - 发现初稿引用的 `platform.openai.com` 文档在本地网络无法访问，**替换**为可直达的 Anthropic Messages API 文档；
   - 发现初稿写错了一个 Anthropic 工程博客的路径，**修正**为 `engineering/effective-context-engineering-for-ai-agents`；
   - Simon Willison 的文章本地 curl 被拦截，但通过网页抓取确认了页面真实存在且内容与引用论断相符，予以保留。
2. **删除了不实的引用**：初稿曾有一条来源标注"支撑多轮对话机制"但页面实际不覆盖该内容，改为让论断与来源一一对应。
3. **改写概念解释**：所有"一句话个人解释"均要求用自己的话组织，与来源原文无连续重合；正文中的引用论断都标注了对应来源。
4. **核对技术表述**：核对 Workflow/Agent 分界（Anthropic《Building Effective Agents》）、渐进式披露机制（Anthropic Skills 文档）、token 估算（中文约 1 字 ≈ 1–2 token）等关键表述与来源一致。
5. **补充个人判断**：每份资料"使用边界"一节添加了"我的判断标准"，将通用知识转化为个人可操作的决策规则。
6. **安全检查**：确认仓库中无 API Key、密码、个人隐私信息；`.gitignore` 排除敏感文件模式。

> 说明：三个概念的选择与材料结构按课程要求设定；SKILL.md 中的流程设计参考了 Anthropic 官方 Skills 文档的命名与结构建议，但学习框架（学习目标/核心问题/自测题等）为个人设计。

## 遇到的问题与最终解决方式（过程记录）

1. **本机没有 `gh` CLI、git 无已存凭据**，无法直接创建远程仓库 → 采用 Fine-grained/Classic PAT 调用 GitHub API 创建仓库，再推送；
2. **`platform.openai.com` 在本地网络无法访问（curl 超时）** → 将该引用替换为可直达验证的 Anthropic Messages API 文档，保证每条来源都可核查；
3. **`git fetch` 后远程跟踪分支 `origin/main` 始终为空**（本地命令环境异常中断导致 ref 未写入）→ 诊断后改用 `git config branch.main.remote/merge` 手工配置上游跟踪，推送与同步恢复正常；
4. **`raw.githubusercontent.com` 偶发超时** → 用 GitHub Contents API（`/repos/.../contents/...`）复核，确认 SKILL.md 等文件均已正确推送；
5. **Token 安全**：推送命令中临时携带 Token，推送后立即将远程 URL 重置为不含凭据的干净地址；提交与文件中扫描确认无任何 Token/密钥残留，Token 用完即应从 GitHub 账户删除；
6. **提交作者信息**：初版提交误用了本地占位邮箱，已修正为 GitHub 账号关联的 noreply 邮箱。

## 来源一览

全部来源链接在各 HTML 文件末尾的"参考来源"章节，均已验证可访问（2026-09-06）。主要包括：

- Anthropic — [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) / [Agent Skills Overview](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) / [Context Windows](https://docs.claude.com/en/docs/build-with-claude/context-windows)
- Lilian Weng — [LLM Powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/)
- Liu et al. — [Lost in the Middle](https://arxiv.org/abs/2307.03172)
- Simon Willison — [Claude Skills](https://simonwillison.net/2025/Oct/16/claude-skills/)
