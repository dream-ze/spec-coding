# 个人跨 Agent Spec Coding 设计规格

状态：等待用户审阅  
日期：2026-08-26

## 1. 目标

建立一套个人使用、同时适用于 Codex 和 Claude、且不依赖单一 Agent 的 Spec Coding 工作流。

它必须做到：

- 功能开发和复杂缺陷必须走完整 Spec；
- 简单修复和配置调整可以走快速通道；
- 用户可以显式选路，Agent 也可以根据风险自动升级；
- Codex 与 Claude 共享同一批项目上下文和规格产物；
- 默认不把个人工作流文件提交进公司仓库；
- 复用 GitHub Spec Kit 的 SDD 和缺陷处理能力，不重复实现底层流程。

## 2. 已确认决策

1. 这是个人工作流，不是团队制度或 CI 门禁。
2. 同时覆盖 Codex 和 Claude。
3. 功能开发、复杂修复必须走 Spec。
4. 简单修复、配置调整允许快速通道。
5. 入口采用混合判断：
   - 用户可以显式要求 `spec-first` 或 `quick-fix`；
   - 未指定时，由 Agent 分类并说明依据；
   - 用户或 Agent 都可以把任务升级为 Spec；
   - 已进入 Spec 的任务不得被静默降级。
6. 项目产物放在当前项目内，让两个 Agent 都能读取。
7. 默认通过 `.git/info/exclude` 做个人本地排除，不修改仓库的 `.gitignore`。
8. 用一个很薄的个人编排 Skill 控制流程，底层委托给 GitHub Spec Kit。

## 3. 总体架构

### 3.1 可移植安装包

首版交付一个可移植的 `spec-coding` 包：

```text
spec-coding/
├── README.md
├── skill/
│   ├── SKILL.md
│   └── references/
│       ├── classification.md
│       ├── quick-path.md
│       └── spec-path.md
├── scripts/
│   ├── install.py
│   └── bootstrap_project.py
└── tests/
    ├── test_package.py
    └── scenarios/
```

`install.py` 将同一份 Skill 分别复制到 Codex 和 Claude 的个人 Skill 目录。采用复制而不是跨产品符号链接，以避免依赖未验证的链接兼容性。再次运行安装器时，两份副本会从同一源目录同步更新。

安装器本身不安装第三方软件、不初始化项目，也不修改项目文件。

### 3.2 统一编排 Skill

`spec-coding` 是个人唯一入口，只负责四件事：

1. 阅读足够的项目上下文并判断任务类型；
2. 输出 `Quick`、`Feature Spec` 或 `Complex Bug Spec`，同时给出简短依据；
3. 强制执行对应路径的审批关卡；
4. 在 Spec Kit 可用时调用它维护的具体工作流。

它不会重新编写 Spec Kit 已提供的规格、方案、任务、分析、实现、收敛或缺陷报告模板。

### 3.3 Spec Kit 项目集成

`bootstrap_project.py` 只在用户明确批准后准备指定项目。它将：

1. 确认目标路径确实是预期项目根目录；
2. 检查 `specify` 是否存在；
3. 初始化或更新 Codex 集成到 `.agents/skills`；
4. 安装 Claude 集成到 `.claude/skills`；
5. 可选安装官方 Bug 扩展；
6. 将个人路径写入 `.git/info/exclude`，不修改 `.gitignore`；
7. 报告全部新增或变更路径。

Codex 和 Claude 是同一批项目产物之上的两个适配器。不同 Agent 的调用语法封装在 Skill 引用文件里，不要求用户记忆两套命令。

## 4. 分类规则

### 4.1 Quick 快速通道

只有同时满足以下条件才能走 Quick：

- 修改的是已有且已理解的流程，而不是新增能力；
- 变更范围局部，且存在明确验证方法；
- 不改变公共 API、持久化数据结构、数据库 Schema、权限规则或跨服务契约；
- 不涉及未知根因、间歇性故障、数据丢失、安全影响或广泛性能风险；
- 阅读项目后没有发现更大的上下游影响。

典型例子包括：文案错误、局部配置值纠正、已有明确复现的局部空值处理、因既定行为变化而调整单个测试预期。

文件数量只作为风险信号，不作为唯一判断标准。

### 4.2 Feature Spec 功能路径

新增或明显改变行为、引入新接口或集成、影响多层调用链，或需要产品与架构决策时，必须走 Feature Spec。

### 4.3 Complex Bug Spec 复杂缺陷路径

根因未知、问题间歇出现或仅在线上出现、可能涉及多个模块，或者影响安全、数据一致性、并发、可靠性、显著性能时，必须走 Complex Bug Spec。

### 4.4 升级规则

Quick 过程中一旦出现隐藏复杂度，Agent 必须先停止继续实现，说明新增证据并建议升级。已经产生的修改保持可见，未经允许不得擅自丢弃。

## 5. 三条执行流程

### 5.1 Quick

```text
阅读项目
→ 宣布 Quick 分类及依据
→ 给出简短设计和验证方法
→ 用户确认
→ 实施局部修改
→ 执行针对性验证
→ 报告修改文件、验证证据和剩余风险
```

Quick 默认不生成持久 Spec 文件，但仍必须在修改代码前获得简短设计批准。

### 5.2 Feature Spec

```text
阅读项目和现有行为
→ 建立或更新 spec
→ 澄清歧义
→ 用户确认需求
→ 生成实现方案
→ 生成有依赖顺序的 tasks
→ 分析各产物的一致性
→ 用户确认实现范围
→ 一次实现一个任务或明确阶段
→ 验证后再继续
→ 对照 spec / plan / tasks 做 converge
→ 最终验收报告
```

较大需求必须按 Task 或明确阶段实现，不允许无范围地一次执行全部任务。

### 5.3 Complex Bug Spec

```text
复现或收集证据
→ 生成只读 assessment
→ 用户确认修复方案
→ 实施有范围的修复
→ 重跑复现步骤和回归测试
→ 记录 verified / partial / failed
```

存在 Spec Kit Bug 扩展时直接委托给官方流程；不可用时，Skill 使用本地 Markdown 产物维持同样的三阶段契约，并明确标注为后备流程。

## 6. 项目产物与本地排除

功能产物使用 Spec Kit 的项目内 `specs/` 约定；缺陷产物使用官方 `.specify/bugs/` 约定。Agent 集成还可能生成 `.specify/`、`.agents/skills/speckit-*` 和 `.claude/skills/speckit-*`。

在 Git 仓库中，bootstrap 只把必要的个人路径加入 `.git/info/exclude`：保留既有内容、避免重复。如果目标路径已经被 Git 跟踪，脚本必须明确报告，不能声称本地排除已经生效。

在非 Git 目录中，产物仍保存在项目内，但脚本会提示不存在本地 Git 排除机制。

只有用户明确要求暂存或提交时，个人规格才会转为仓库交付物；工作流绝不自动完成这个动作。

## 7. 异常与安全处理

- 缺少 `specify`：说明依赖缺失，提供内置 Markdown 后备流程或显式安装选项，禁止自动安装。
- 工作树已有修改：保留无关变更；初始化或实现前先报告重叠目标。
- 已安装 Spec Kit：先读取集成状态和计划变更，得到批准后才原位升级。
- 已存在同名 Skill：停止并要求用户选择改名、备份或替换。
- bootstrap 部分失败：报告已完成操作和恢复方式；回滚时不删除用户文件。
- 分类不明确：选择更严格的 Spec 路径。
- 验证失败：只能报告 `partial` 或 `failed`，不能声称完成。

## 8. 测试策略

### 8.1 包级自动测试

- 校验 `SKILL.md` frontmatter 和必需章节；
- 校验所有引用文件都存在；
- 校验 Codex、Claude 安装副本与源 Skill 完全一致；
- 校验重复安装具备幂等性；
- 校验 bootstrap 不重复写入、不破坏 `.git/info/exclude` 其他内容；
- 校验 dry-run 准确列出计划修改；
- 校验已跟踪路径与非 Git 目录的告警。

### 8.2 分类场景测试

- 新增接口 → Feature Spec；
- 线上间歇性超时 → Complex Bug Spec；
- 局部配置值纠正 → Quick；
- Quick 中发现 Schema 变化 → 升级为 Feature Spec；
- 对微小修改显式要求 `spec-first` → 保持 Spec；
- 对认证逻辑显式要求 `quick-fix` → 拒绝 Quick 并升级。

### 8.3 人工冒烟测试

在 Codex 和 Claude 中分别运行同一个代表性需求，确认两者输出相同分类、执行相同审批关卡，并读取同一批项目产物。

## 9. MVP 验收标准

1. 一个安装包能把同一份 `spec-coding` Skill 安装给 Codex 和 Claude。
2. 两个 Agent 都能显式调用，并能通过相关自然语言需求自动匹配。
3. 场景矩阵得到一致分类。
4. Quick 在简短设计获批前不能进入实现。
5. Feature 和 Complex Bug 在所需产物与审批齐备前不能进入实现。
6. Codex、Claude 的 Spec Kit 集成可以同时存在于临时测试仓库。
7. 个人项目产物能在不修改 `.gitignore` 的情况下本地排除。
8. 重复安装和 bootstrap 安全且幂等。
9. 自动测试全部通过，两个 Agent 的人工冒烟结果有记录。

## 10. MVP 非目标

- 团队或组织级规则；
- Git Hook、pre-commit 或 CI 阻断；
- 自动 commit、push 或创建 PR；
- 自动安装第三方依赖；
- 支持 Codex、Claude 之外的 Agent；
- 自研一套替代 Spec Kit 的 SDD 引擎。

## 11. 首次实现边界

第一版只生成可移植安装包，并在临时测试仓库中验证。它不会直接安装进用户当前的 Codex、Claude 配置，也不会初始化任何现有公司仓库。真实安装属于审阅原型之后的独立显式操作。
