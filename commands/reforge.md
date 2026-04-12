---
description: 回炉再造 — verify pre → 4 agent 审查 → 美容 → verify post，注入项目规则和踩坑记录，自循环直到连续 3 次 CLEAN
---

# /reforge — 回炉再造

## Phase 0：开工前检查（verify --pre）

1. 检测 cwd：home 目录用 `verify.py`，auto-trading 用 `wuji-verify.py`
2. 运行：
   ```
   # 太极域
   python3 ~/.claude/merit/verify.py --pre <files...>
   # 两仪域
   python3 ~/.claude/merit/wuji-verify.py --pre <files...>
   ```
3. 输出踩坑提醒 + 当前反模式状态 + 文件文档绑定
4. 如果有 ❌ 级违规，先修再进入 Phase 1

## Phase 1：审查修复（自动循环）

### 准备

1. 确定审查范围：
   - 如果太极指定了模块路径（如 src/core/）→ 扫该目录下所有 .py
   - 如果用户指定了文件列表 → 用指定的
   - 如果都没指定 → 扫整个项目所有 .py
   - **禁止自行缩小范围**。太极给了模块就全扫，不能挑几个文件宣布 CLEAN
2. 用 Bash 运行上下文注入引擎：
   ```
   python3 ~/.claude/merit/reforge_context.py <file1> <file2> ...
   ```
3. 将输出作为 `PROJECT_CONTEXT` 变量

### 执行

并行启动 4 个 Agent（subagent_type: Explore, model: sonnet），每个 agent 的 prompt 末尾附加 `PROJECT_CONTEXT`：

**Agent 1 — 复用检查：**
- 重复代码（5+ 行相同的块）
- 可共享的函数（多处内联同一逻辑）
- 重复常量定义

**Agent 2 — 质量检查：**
- 死代码（未使用变量/不可达分支/废弃 import）
- 命名不清（变量/函数名不描述用途）
- 冗余状态（可推导的缓存值）
- 不必要的注释（描述显而易见的代码）

**Agent 3 — 效率检查：**
- 热路径重复 I/O（同一文件多次读取）
- N+1 模式
- TOCTOU 反模式（先检查存在再操作）
- 资源泄漏（未关闭的文件/连接）

**Agent 4 — 安全+正确性检查：**
- SQL/Shell/路径注入
- 并发竞态（无锁写入共享文件）
- 边界条件（None/空/溢出/off-by-one）
- 对照 PROJECT_CONTEXT 中的 ANTI_PATTERNS 和项目规则

### 循环

- 4 个 agent 全部返回后，汇总 findings
- 有问题 → 逐个修复 → 重新启动 4 个 agent
- 无问题 → `clean_count += 1`
- **连续 3 次 CLEAN → 进入 Phase 2**
- **最多 10 轮**，超过则停止并报告

## Phase 2：美容（自动循环）

启动 1 个 Agent（subagent_type: Explore, model: sonnet），**不注入项目上下文**（纯结构改善）：

- 不必要的嵌套 → 用 early return 扁平化
- 重复块 → 提取 helper
- 复杂表达式 → 拆成命名变量
- 命名改善

### 循环

- 有问题 → 修复 → 重跑
- **连续 3 次 CLEAN → 完成**
- **最多 10 轮**

## Phase 3：交活前检查（verify --post）

1. 运行：
   ```
   # 太极域
   python3 ~/.claude/merit/verify.py --post <files...>
   # 两仪域
   python3 ~/.claude/merit/wuji-verify.py --post <files...>
   ```
2. 对比 --pre 时的违规数，不能比开工前多
3. 如果有新增违规 → 修复 → 重跑 --post

## 完成

输出摘要：
```
🔥 回炉再造完成
   Phase 0 verify pre：✅ 通过
   Phase 1 审查：X 轮（修复 N 项）
   Phase 2 美容：Y 轮（修复 M 项）
   Phase 3 verify post：✅ 无新增违规
   总计：X+Y 轮，全部 CLEAN
```

## 规则

- 保持功能不变 — 只改 HOW 不改 WHAT
- 每次修复后 py_compile 验证
- 修复时参照 PROJECT_CONTEXT 中的规则和踩坑记录
- 如果 finding 是误报或不值得改，跳过不争论
