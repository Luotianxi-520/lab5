# CampusTask — 需求分析与迭代设计

基于 3 名用户的访谈，收集并整理需求，从中选择 2 个高优先级需求实现。

## 用户访谈总结

本次访谈了 3 名计算机专业学生，了解他们在课程任务管理中的具体需求：

| 用户 | 身份 | 核心痛点 |
|------|------|----------|
| 用户A | 大三学生 | 同时跟 6 门课，任务多，需要导出清单打印或分享给学习小组 |
| 用户B | 大二学生 | 经常忘记截止日期，希望按时间排序知道哪些最紧迫 |
| 用户C | 研一学生 | 需要区分优先级，希望看到 overdue 的任务 |

## 需求清单

| # | 功能 | 优先级 | 验收标准 | 用户原话 | 预估工作量 | 风险 |
|---|------|--------|----------|----------|------------|------|
| 1 | 截止日期字段 | P0 | `add --deadline YYYY-MM-DD` 创建带日期的任务 | "我怕忘记交作业"（用户B） | 小（已实现于 v0.2） | 低：字段可选，不影响现有逻辑 |
| 2 | 优先级字段 | P0 | `add --priority high\|medium\|low` | "我想知道哪些最重要"（用户C） | 小（已实现于 v0.2） | 低：字段可选，枚举值可控 |
| 3 | 按截止日期排序 | P1 | `list --sort deadline` 按日期升序排列 | "我想一眼看出哪个最先截止"（用户B） | 小 | 低：纯排序逻辑，无副作用 |
| 4 | 导出为 CSV | P1 | `export tasks.csv` 生成可打开的 CSV 文件 | "我想打印出来贴桌上或发给同学"（用户A） | 小 | 低：标准库 csv 模块，UTF-8 BOM 确保 Excel 兼容 |
| 5 | 按优先级排序 | P1 | `list --sort priority` 按 high→medium→low 排列 | "我想优先做重要的事"（用户C） | 小 | 低：纯排序逻辑，无副作用 |
| 6 | 逾期提醒 | P2 | `list --overdue` 显示已过期的待办任务 | "过了截止日期还不知道就糟糕了"（用户C） | 小 | 中：依赖系统时间，需与 deadline 字段协同，时区差异可能导致误判 |

### 选择理由

本次迭代选择实现需求 #3（按截止日期排序）、#4（导出 CSV）、#5（按优先级排序）和 #6（逾期提醒），理由：
- deadline 和 priority 字段（P0）已在 v0.2 完成，排序、导出和逾期是对已有字段的自然延伸
- 四个需求均预估工作量小，可在一次迭代中完成
- 逾期提醒从 P2 backlog 提升到本迭代，因为依赖的 deadline 字段已稳定

## 新增功能

### 导出 CSV

```bash
python main.py export 任务清单.csv
# 输出: 已导出 5 条任务到 任务清单.csv
```

生成的 CSV 文件包含列：`id, title, status, created_at, deadline, priority`，使用 UTF-8 BOM 编码，Excel 可直接打开。

### 按截止日期排序

```bash
python main.py list --sort deadline
# 有截止日期的任务按时间升序在前
# 无截止日期的任务排在末尾

python main.py list --status todo --sort deadline
# 与状态过滤组合使用
```

### 按优先级排序

```bash
python main.py list --sort priority
# 按 high → medium → low 排列
# 无优先级的任务排在末尾

python main.py list --status todo --sort priority
# 与状态过滤组合使用
```

### 查看逾期任务

```bash
python main.py list --overdue
# 显示已过截止日期且仍未完成的待办任务
```

## 使用方法

```bash
# 添加任务
python main.py add "提交实验报告" --deadline 2026-06-30 --priority high

# 按截止日期排序查看
python main.py list --sort deadline

# 按优先级排序查看
python main.py list --sort priority

# 查看逾期任务
python main.py list --overdue

# 查看待办任务（按日期排序）
python main.py list --status todo --sort deadline

# 导出
python main.py export 任务清单.csv

# 其他命令
python main.py done 1
python main.py delete 2
python main.py edit 1 "修改后的标题"
python main.py search "实验"
python main.py stats
```

## 数据流

```
用户输入 → cli.py 解析参数 → task_manager.py 业务处理 → storage.py JSON 读写
                                                              ↓
                                    export_csv() ──→ CSV 文件输出
```

## 测试

```bash
.venv/Scripts/python -m pytest tests/ -v
# 39 passed in 0.84s
```

## 实验反思

**问题：为什么"用户说想要什么"和"系统该做什么"不是一回事？**

用户A 说"我想打印出来贴桌上"——她真正的需求不是打印，而是脱离命令行也能查看任务。如果直接实现"打印"功能（调用系统打印 API），反而做了一个没人用的功能（学生不需要纸质版，同学分享用截图就够了）。正确的做法是把需求翻译成"导出为常见格式"——CSV 可以用 Excel 打开查看、可以打印、可以发到群里。这是从"用户说了什么"到"系统该有什么能力"的翻译。

用户B 说"我想一眼看出哪个最先截止"——他不需要一个新界面，只需要列表按日期排序。这个需求如果不加约束，可能被过度开发（做日历视图、做 Gantt 图），但用 `--sort deadline` 一个参数就解决了。

总结：需求分析的本质不是记录用户的话，而是识别用户的真正问题，然后用最小可行方案解决它。用户说的"我要 X"往往只是解决方式之一，工程师需要回到问题本身："你遇到什么困难了？"
