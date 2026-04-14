---
name: task-summary-logging
description: Use after completing any task or multi-step operation to capture executed commands, results, and lessons learned for future reference
---

# Task Summary Logging

After completing a task, summarize what was done and save it as a timestamped Markdown file for future reference.

## Workflow

```
Task complete?
  → Compile summary
  → Show to user for review/approval
  → User approves?
    → Yes: Save to file
    → No: Revise based on feedback
```

## Format

File name: `~/.claude/task-logs/yyyyMMdd-HHmm-任务名称.md`

```markdown
# 任务名称

**日期:** YYYY-MM-DD HH:mm
**耗时:** 约 X 分钟

## 目标
[一句话说明要做什么]

## 操作记录

### 步骤1: xxx
```bash
执行的命令
```
结果：成功/失败 + 关键输出摘要

### 步骤2: xxx
...

## 经验教训

- ✅ 有效做法：xxx
- ⚠️ 注意事项：xxx
- ❌ 踩过的坑：xxx

## 相关文件

- `路径1` — 说明
- `路径2` — 说明
```

## Common Mistakes

- **没等用户确认就保存** — 必须先把总结展示给用户，确认后再保存
- **记录过于冗长** — 命令只记关键的，不要每一步都记；结果只记关键输出，不全贴
- **经验教训空着** — 哪怕只有一条"scp 不可用时用 ssh cat 替代"也值得记
- **文件名含特殊字符** — 用日期前缀保证排序，任务名用中文没问题，但避免 `/ \ : * ? " < > |`
