token：ada_9f1c1d0474544997aedd503a347a4404



## CC 工作原理

### Agentic Loop（三阶段循环）

CC 执行任务的核心是一个循环：**收集上下文 → 采取行动 → 验证结果**，三个阶段反复迭代直到任务完成。

- 简单问题：只需收集上下文
- Bug 修复：三个阶段反复循环
- 大型重构：以验证为主的深度循环

每个工具调用的结果都会反馈到下一步决策，这是 agentic 的核心。

### 内置工具分类

| 类别           | 能做什么                                           |
| -------------- | -------------------------------------------------- |
| File 文件操作  | 读文件、编辑代码、创建/重命名文件                  |
| Search 搜索    | 按 pattern 查找文件、正则搜索内容                  |
| Execution 执行 | 运行 shell 命令、启动服务、跑测试、git 操作        |
| Web            | 搜索网页、抓取文档、查错误信息                     |
| Code 智能      | 查看类型错误、跳转定义、查找引用（需安装插件）     |

> 还有 subagent、提问等编排类工具，详见官方 Tools Reference

### CC 可访问的内容

启动后 CC 能访问：项目目录文件、终端（任何你能跑的命令它也能）、当前 git 状态、CLAUDE.md 指令、Auto Memory 记忆、配置的 MCP/Skills/Subagents

### Session 管理

- 每条消息、工具调用结果都本地存在 `~/.claude/projects/` 的 JSONL 文件里
- **每个 session 上下文独立**，不继承上次对话历史
- 跨 session 持久化靠 Auto Memory 和 CLAUDE.md

| 操作               | 命令/行为                                       |
| ------------------ | ----------------------------------------------- |
| 恢复上次 session   | `claude --continue`                             |
| 选择历史 session   | `claude --resume`                               |
| Fork 一个新 session | `/branch`，复制历史到新 ID，原 session 不受影响 |
| 并行多个 session   | 用 git worktrees 创建独立目录                   |

### 上下文窗口管理

上下文包含：对话历史、文件内容、命令输出、CLAUDE.md、Auto Memory、Skills 内容

- CC 会自动 compact：先清理旧的工具输出，再摘要对话
- 在 CLAUDE.md 里加 `Compact Instructions` 区块，可控制压缩保留什么
- Skills 按需加载（用到时才把全文加载进来），Subagents 有独立上下文不影响主窗口

### Checkpoint 文件回滚

每次文件编辑前 CC 会自动快照原文件，按 **ESC×2** 可回退到上一个状态。
> Checkpoint 只覆盖文件变更，不覆盖数据库/API/部署等远端操作。



---

### 如何用好CC

核心在于：委托目标，而不是指挥步骤 告诉它你的需求



上下文管理:

| 命令                  | 作用                              |
| --------------------- | --------------------------------- |
| /context              | 查看当前上下文                    |
| /compact              | 手动压缩对话历史                  |
| /compact focus on xxx | 压缩但保留指定焦点                |
| /clear                | 清空上下文(新任务用)              |
|                       |                                   |
| /resume               | 打开历史会话选择器 可恢复上次对话 |
| /model                | 切换模型                          |
|                       |                                   |
| /plan                 | 进入只读分析模式，先规划 不改代码 |



| 命令         | 作用                             |
| ------------ | -------------------------------- |
| /init        | 自动分析项目 生成CLAUDE.md       |
| /memory      | 查看和编辑持久记忆文件           |
| /permissions | 查看当前权限配置                 |
|              |                                  |
| /code-review | 默认medium深度 可加--fix直接修复 |



启动参数：

claude --continue恢复最近一次会话 --resume选择历史会话 -



权限控制 shift+tab切换

| 模式              | 说明             | 适用场景 |
| ----------------- | ---------------- | -------- |
| default           | 每次操作都询问   |          |
| acceptEdits       | 自动批准文件编辑 |          |
| plan              | 只读             |          |
| bypassPermissions | 全部跳过         |          |



Memory系统

CC有三层持久化，重要性依次降低

1. CLAUDE.md 每次对话自动注入 项目级
2. Memory文件  用户级
3. git文件

主动让Claude记住 





好的使用TIPS：

"/plan 然后帮我梳理一下 /api/trace/getAnalysisResultNew 的完整调用链"

 "/plan 我想在 label 过滤里支持 bu_name 参数，你打算怎么改"

"/compact focus on SQLGenerateService 的 AB 标签 JOIN逻辑“



| 快捷键    | 作用                                  |
| --------- | ------------------------------------- |
| ESC       | 中断当前操作                          |
| shift+tab | 循环切换权限模式                      |
| ⬆️         | 回调上一条输入                        |
| ！<命令>  | 直接在会话中  ！git log --oneline -10 |



