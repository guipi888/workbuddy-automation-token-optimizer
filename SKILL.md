---
name: "automation-token-optimizer"
summary: "自动化定时任务 Token 优化审计工具 —— 每月自动扫描所有定时任务的 token 消耗，找出高消耗任务并给出脚本化/降频/精简的具体优化方案"
read_when:
  - 用户问"哪些定时任务消耗 token 最多"
  - 用户想优化自动化任务的 token 成本
  - 用户需要每月自动审计定时任务体系
---

# 自动化定时任务 Token 优化

每月自动审计所有定时任务的 token 消耗情况，给出可执行的优化方案。

## 适用场景

- 定时任务越来越多，想控制 token 成本
- 想知道哪些任务可以改成纯脚本执行（不消耗 AI token）
- 需要定期的自动化任务效率报告

## 核心功能

### 1. Token 消耗统计

自动读取 `workbuddy.db`，统计上一个自然月每个定时任务的：
- 运行次数
- 输入 token 总量
- 输出 token 总量
- 单次平均消耗

按总消耗从高到低排序，找出 top 5 高消耗任务。

### 2. Prompt 分析

逐一查看高消耗任务的 prompt 内容，判断：
- 是否有纯数据拉取/格式化环节（可改为脚本）
- 是否有重复执行但结果相似的步骤（可缓存）
- 是否 prompt 过长（可精简）

### 3. 优化建议

针对每个高消耗任务，给出具体优化方向：

| 优化方向 | 说明 | 预期收益 |
|---------|------|---------|
| 🔧 **改为脚本** | 纯数据拉取/格式化类，改用 Python 脚本直接输出 | 节省 100% AI token |
| ⏰ **降频** | 每日改每周或触发式 | 节省 80%+ token |
| ✂️ **精简 prompt** | 去掉冗余描述，减少 context 长度 | 节省 20-40% token |
| 🔀 **拆分任务** | 数据拉取和 AI 分析拆成两个任务 | 数据拉取部分节省 100% token |

### 4. 审计报告格式

```markdown
## 📊 月度定时任务 Token 审计报告（YYYY-MM）

### 本月概览
- 自动化任务总运行次数：X 次
- 总 token 消耗：XXX 万 tokens（输入 / 输出）
- 高消耗任务（前 5）

### 🔴 高消耗任务排行
| 排名 | 任务名 | 月运行次数 | 月消耗 token | 每次均值 | 优化建议 |
|------|--------|-----------|-------------|---------|---------|
| 1 | ... | ... | ... | ... | ... |

### 💡 优化行动建议（可执行）
1. **[任务名]**：建议 → 具体操作步骤
2. ...

### ✅ 结论
当前定时任务体系消耗是否在合理范围？哪些任务性价比最高？哪些可以优化或关停？
```

## 使用方法

### 一次性分析

直接对 AI 说："帮我分析一下现在所有定时任务的 token 消耗情况，哪些可以脚本化？"

AI 会：
1. 列出所有定时任务
2. 读取 workbuddy.db 统计 token 消耗
3. 分析高消耗任务的 prompt
4. 给出优化建议

### 定期自动审计

设置一个每月1号执行的定时任务，prompt 内容如下：

```
你是一个定时任务审计专家。每月 1 号上午执行，目标是排查本月所有定时任务的 token 消耗情况，给出优化方案和结论。

## Step 1：获取所有定时任务

用 automation_update 工具（mode="list"）列出所有当前定时任务，记录：
- 任务 ID、名称、状态（ACTIVE/PAUSED）
- 执行频率（rrule / scheduledAt）
- 工作目录

## Step 2：查 token 消耗数据

读取 workbuddy.db，分析上一个自然月（第一天到最后一天）每个任务的 token 消耗：

```bash
sqlite3 ~/.workbuddy/workbuddy.db "
SELECT 
  s.title,
  s.id,
  COUNT(*) as run_count,
  SUM(su.input_tokens) as total_input,
  SUM(su.output_tokens) as total_output,
  SUM(su.input_tokens + su.output_tokens) as total_tokens,
  ROUND(AVG(su.input_tokens + su.output_tokens)) as avg_per_run
FROM sessions s
LEFT JOIN session_usage su ON s.id = su.session_id
WHERE s.is_background_automation = 1
  AND s.created_at >= strftime('%s', 'now', 'start of month', '-1 month') * 1000
  AND s.created_at < strftime('%s', 'now', 'start of month') * 1000
GROUP BY s.title
ORDER BY total_tokens DESC;
"
```

同时获取总消耗：

```bash
sqlite3 ~/.workbuddy/workbuddy.db "
SELECT 
  COUNT(DISTINCT s.id) as total_runs,
  SUM(su.input_tokens) as total_input,
  SUM(su.output_tokens) as total_output,
  SUM(su.input_tokens + su.output_tokens) as grand_total
FROM sessions s
LEFT JOIN session_usage su ON s.id = su.session_id
WHERE s.is_background_automation = 1
  AND s.created_at >= strftime('%s', 'now', 'start of month', '-1 month') * 1000
  AND s.created_at < strftime('%s', 'now', 'start of month') * 1000;
"
```

## Step 3：分析每个任务的 prompt

用 automation_update 工具（mode="view"）逐一查看排名前 5 高消耗任务的 prompt 内容，重点看：
- 是否有可以改为本地脚本/API 直接调用的环节
- 是否有重复执行但结果相似的步骤
- 是否可以降低执行频率而不影响效果

## Step 4：给出优化建议

针对每个高消耗任务，给出具体优化方向，例如：
- 🔧 **改为脚本**：纯数据拉取/格式化类任务，可改为 Python 脚本直接输出，不走 AI 推理
- ⏰ **降频**：某些任务可以从「每日」改为「每周」或「触发式」
- ✂️ **精简 prompt**：去掉冗余描述，减少 context 长度
- 🔀 **拆分任务**：把「数据拉取」和「AI 分析」拆成两个独立任务，前者用脚本，后者按需触发

## Step 5：输出审计报告

输出格式：

---
## 📊 月度定时任务 Token 审计报告（YYYY-MM）

### 本月概览
- 自动化任务总运行次数：X 次
- 总 token 消耗：XXX 万 tokens（输入 / 输出）
- 高消耗任务（前 5）

### 🔴 高消耗任务排行
| 排名 | 任务名 | 月运行次数 | 月消耗 token | 每次均值 | 优化建议 |
|------|--------|-----------|-------------|---------|---------|
| 1 | ... | ... | ... | ... | ... |

### 💡 优化行动建议（可执行）
1. **[任务名]**：建议 → 具体操作步骤
2. ...

### ✅ 结论
当前定时任务体系消耗是否在合理范围？哪些任务性价比最高？哪些可以优化或关停？

---

工作目录：<你的 WorkBuddy 工作目录>（首次使用时 AI 会主动询问并确认）
```

## 技术细节

### 数据库表结构

- `sessions` 表：`is_background_automation = 1` 表示定时任务触发的会话
- `session_usage` 表：`input_tokens` 和 `output_tokens` 字段记录每次会话的 token 消耗
- 时间字段 `created_at` 是毫秒级时间戳

### 依赖工具

- `automation_update` 工具：列出/查看定时任务
- `sqlite3` 命令行：读取 workbuddy.db
- `workbuddy.db` 路径：`~/.workbuddy/workbuddy.db`

## 注意事项

1. **数据库路径可能因环境不同而变化**，建议优先使用 `~/.workbuddy/workbuddy.db`
2. **SQLite 时间函数**：`strftime('%s', ...)` 返回秒级时间戳，需要乘以 1000 转为毫秒
3. **优化建议需要人工确认**：脚本化改造可能需要调整业务逻辑，建议先在小范围测试

## 版本迭代记录

| 版本 | 日期 | 更新内容摘要 | 操作人 |
|------|------|------------|--------|
| v1.0 | 2026-06-20 | 从定时任务 prompt 打包为可发布 Skill | Kyle |
