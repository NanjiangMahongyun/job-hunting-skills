---
name: memory-manager
description: 记忆库管理skill。负责读取、更新、归档用户的持久化记忆（profile/career-timeline/current-state/analysis-archive），确保跨会话信息不丢失。每次对话开始时自动读取记忆库，重大变化时更新，分析结果归档。触发词：记住、更新档案、记录、归档、上次的情况、我之前的分析。
---

# 记忆库管理（跨会话持久化）

## 这个 skill 干什么

解决"过段时间回来找不到之前的交互记录"的问题。把用户的关键信息、分析结果、职业变化持久化到本地文件，下次对话直接读取接上。

**记忆库位置**：`memory/` 目录。**公开仓库里不放这个目录，也不放空目录占位**（git 不记录空目录，提交进去也没有效果）——目录由本 skill 在**第一次要写文件时**创建：`mkdir -p memory/analysis-archive memory/job-pool/history`。

- `profile.md` — 用户基本档案（技术栈、项目、偏好、人际关系）
- `career-timeline.md` — 职业时间线（入职/离职/求职，按时间倒序）
- `current-state.md` — 当前状态快照（职业状态、求职进度、待办）
- `resume-YYYY-MM.md` — 简历各版本
- `analysis-archive/` — 一次性分析归档（**append-only：旧版不覆盖、不删除，重新分析另存新日期版本**）
- `job-pool/` — 岗位池**当前工作集**（会被反复增删，允许覆盖，但覆盖前必须把旧版另存进 `job-pool/history/`）

**两类文件的区别是生命周期，不是主题**：尽调、谈薪、行业分析是"一次分析出一份结论"→ 进 `analysis-archive/` 存档；岗位池是"随补搜不断改写的活文档"→ 进 `job-pool/`。把活文档塞进 append-only 的归档目录，只会逼人要么违反"不覆盖"、要么每次手抄一份。

### 生产文件落点总表（本仓库唯一的落点权威，其他 skill 以此为准）

| 产出方 | 落点 | 文件名 | 覆盖规则 |
|---|---|---|---|
| company-due-diligence | `memory/analysis-archive/` | `YYYY-MM-DD-公司尽调-[公司名].md` | 只新增 |
| job-negotiation-strategist | `memory/analysis-archive/` | `YYYY-MM-DD-谈判筹码-[公司名].md` | 只新增 |
| salary-negotiation | `memory/analysis-archive/` | `YYYY-MM-DD-谈薪方案-[公司名].md` | 只新增 |
| contract-reviewer | `memory/analysis-archive/` | `YYYY-MM-DD-合同审查-[公司名].md` | 只新增 |
| severance-calculator | `memory/analysis-archive/` | `YYYY-MM-DD-离职补偿-[公司名].md` | 只新增 |
| industry-background-analyzer | `memory/analysis-archive/` | `YYYY-MM-DD-行业分析-[方向].md` | 只新增 |
| job-switch-decision | `memory/analysis-archive/` | `YYYY-MM-DD-跳槽决策-[方向].md` | 只新增 |
| onjob-management | `memory/analysis-archive/` | `YYYY-MM-DD-在岗记录-[主题].md` | 只新增（轻量结论只改 `current-state.md`） |
| jd-analysis-and-resume-tailoring | `memory/resume-YYYY-MM.md` ＋ `profile.md` 的"简历版本"字段 | 按月份命名 | 新版本另存，旧简历不删 |
| job-recommender | `memory/job-pool/` | `[地区]-[方向].md`（如 `南京-AI方向.md`） | **允许覆盖**；覆盖前把旧版另存 `memory/job-pool/history/YYYY-MM-DD-[地区]-[方向].md` |
| skill-upgrade-maintainer | 直接改仓库内文件，不产数据文件 | — | 改动记进 git 提交信息 |

🚨 **不得落在这三处之外**：仓库根目录、`knowledge-base/`、`.claude/skills/`。这三处都属于要公开发布的内容，落进去就等于把真实公司名、真实薪资写进公开库。

**公开性边界（参考依据：`knowledge-base/公开发布与免责合规手册.md`）**：记忆库与 `analysis-archive/` 里是**真实的人、真实的公司、真实的薪资**。它们只留在本地——`/memory/` 已在 `.gitignore` 里，任何"把这包内容发到 GitHub/群里"的动作之前，先按该手册第四节（个人信息）与第九节（发布前自查命令）过一遍：真实主体一律换成虚构示例，而不是打码；他人提供的截图、聊天记录、offer、工资单属于他人个人信息与可能含商业秘密的材料，不对外发布。

---

## 执行流程

### 第0步（前置）：确认求职身份（大学生 / 社会人员）

**读记忆之前先确认身份，写记忆时同步维护身份字段**：

**读取时（🎓 / 💼 都适用）**
- 先检查 `memory/profile.md` 与 `memory/current-state.md` 的**「求职身份」字段是否仍在有效期**——身份会随**毕业、入职、转正、离职**而变化（应届生 → 社会人员的转变尤其常见，且会改变后续所有 skill 的分支）
- 字段缺失、无变更日期、或距今超过 6 个月且期间发生过毕业/入职/离职 → **先向用户确认再继续**，不得直接套用旧身份

**更新时**
- 身份发生变化 → 把新身份写入 `memory/profile.md` 与 `memory/current-state.md`（两处都要），并**记录变更日期**与触发事件（如"2026-07 毕业，🎓 应届生 → 💼 社会人员"）
- 同时在 `memory/career-timeline.md` 留下对应条目

**🎓 大学生 / 应届生 vs 💼 社会人员（本 skill 的身份差异）**
- 🎓：档案重点是学校/专业/毕业时间/实习/校招进度，需额外标注"应届身份失效时间"
- 💼：档案重点是现职公司与在职时长/当前薪资流水/职级/竞业协议状态，需额外标注"社保连续性与背调口径"

**判定规则**：优先直接问用户；用户自述优先于简历/历史记录的任何推断，档案与归档只作证据；无法确定时默认"🎓 大学生"分支，并在报告顶部标注「身份待确认」。

---

### 场景1：每次对话开始（自动执行）

1. 读取 `memory/current-state.md` — 了解用户当前处境
2. 读取 `memory/profile.md` — 了解用户背景和偏好
3. 如果用户提到"上次""之前"，读取 `memory/career-timeline.md` 找对应记录
4. 如果需要上次的分析细节，到 `memory/analysis-archive/` 找对应文件
5. 按第0步（前置）校验「求职身份」字段是否在有效期，过期或缺失则先向用户确认，确认结果再回写档案

### 场景2：用户提供新资料（如简历）

1. 提取关键信息（技能、经历、项目、目标）
2. 更新 `memory/profile.md` 对应字段
3. 如果是新简历，保存到 `memory/` 目录（如 `memory/resume-2026-10.md`）
4. 在 `memory/current-state.md` 更新"简历版本"字段
5. 告诉用户："已更新档案，记录了X条新信息"

### 场景3：完成一次重要分析（如公司尽调、谈薪策略）

1. 将分析结果保存到 `memory/analysis-archive/YYYY-MM-DD-[类型]-[公司名].md`
2. 在 `memory/career-timeline.md` 添加一条记录，关联归档文件
3. 在 `memory/current-state.md` 更新"最近一次分析"字段
4. 告诉用户："分析已归档到 memory/analysis-archive/xxx.md，下次可直接引用"

### 场景4：职业状态变化（入职/离职/拿到offer）

1. 更新 `memory/current-state.md` 的职业状态
2. 在 `memory/career-timeline.md` 添加对应记录
3. 如果是入职，记录薪资、合同条款、关键信息
4. 如果是离职，记录补偿情况、离职证明、后续计划
5. 告诉用户："状态已更新，记录了X"
6. 若本次变化导致**求职身份**改变（毕业、入职、转正、离职转社招），按第0步（前置）同步更新 `profile.md` 与 `current-state.md` 的「求职身份」字段，并**记录变更日期**

### 场景5：用户回来继续上次的事

1. 读取 `memory/current-state.md` 和 `memory/career-timeline.md`
2. 找到上次的分析归档，读取关键结论
3. 问用户："上次分析到X，现在是继续用这个结论，还是根据当前市场重新分析？"
4. 如果用户要求重新分析，用对应skill重新跑，归档新版本

### 场景6：岗位池刷新（job-recommender 产出，或用户手动增删岗位）

1. 目标文件：`memory/job-pool/[地区]-[方向].md`（当前工作集，同一地区＋方向全库只此一份在用的）
2. **要覆盖先留档**：把旧版原样复制为 `memory/job-pool/history/YYYY-MM-DD-[地区]-[方向].md`，再写新版
3. 新版顶部保留检索条件与口径（地区／方向／薪资底线／硬条件是否放宽），下次补搜不用重问
4. 在 `memory/current-state.md` 更新"岗位池：共X个（达标Y＋备选Z），最后刷新日期"
5. 回报："岗位池已更新到 memory/job-pool/xxx.md，旧版在 history/2026-10-21-xxx.md"
6. 🚨 岗位池**不进** `analysis-archive/`（那是 append-only 的一次性结论），也**绝不写到仓库根目录**——根目录是要公开发布的

---

## 归档文件命名规范

```
memory/
├── profile.md
├── career-timeline.md
├── current-state.md
├── resume-2026-10.md
├── analysis-archive/
│   ├── 2026-10-21-公司尽调-XX科技.md
│   ├── 2026-10-21-谈判筹码-XX科技.md
│   ├── 2026-10-21-谈薪方案-XX科技.md
│   ├── 2026-10-21-行业分析-Java后端.md
│   └── 2026-10-21-离职维权-XX科技.md
└── job-pool/
    ├── 南京-AI方向.md                      ← 当前工作集，会反复增删
    └── history/
        └── 2026-10-21-南京-AI方向.md       ← 被覆盖前的旧版
```

归档格式：`YYYY-MM-DD-[分析类型]-[对象].md`；岗位池格式：`[地区]-[方向].md`（不带日期，日期只在 `history/` 里）。

---

## 与其他 skill 的配合

**每个产出分析结果的 skill 跑完都要调用本 skill 归档**，落点见上面「生产文件落点总表」，不要在各自的 SKILL.md 里另写一套路径：

```
company-due-diligence            → analysis-archive/YYYY-MM-DD-公司尽调-[公司名].md
job-negotiation-strategist       → analysis-archive/…-谈判筹码-[公司名].md
salary-negotiation               → analysis-archive/…-谈薪方案-[公司名].md
contract-reviewer                → analysis-archive/…-合同审查-[公司名].md
severance-calculator             → analysis-archive/…-离职补偿-[公司名].md
industry-background-analyzer     → analysis-archive/…-行业分析-[方向].md
job-switch-decision              → analysis-archive/…-跳槽决策-[方向].md
onjob-management                 → analysis-archive/…-在岗记录-[主题].md（轻结论只更新 current-state.md）
job-recommender                  → job-pool/[地区]-[方向].md（覆盖前旧版进 job-pool/history/）
jd-analysis-and-resume-tailoring → resume-YYYY-MM.md ＋ profile.md 的"简历版本"字段
memory-manager（本skill）         → 负责建档与归档，不独立产分析结论
skill-upgrade-maintainer         → 不产数据文件，改动写进 git 提交信息
```

归档完成后必须回报**实际写入的路径**（真写了就给路径，没写就说"本次未落盘"），不得只回一句"已保存记录"。

---

## 禁止行为

1. ❌ 禁止编造用户信息，所有内容必须来自用户提供或分析结果
2. ❌ 禁止覆盖用户手动编辑的内容，更新时保留用户写的部分
3. ❌ 禁止删除历史归档，旧版本保留，新版本另存
4. ❌ 禁止把敏感信息（身份证、银行卡、具体薪资数字）写入 profile.md，这些放归档里并标注"敏感"
5. ❌ 禁止在用户不知情的情况下更新记忆库，重大更新要告诉用户改了什么
6. ❌ 禁止把记忆库内容或真实主体的分析报告带进公开仓库、社群分享或对外文档——需要对外用的是脱敏后的方法与虚构示例
7. ❌ 禁止把生产文件写到 `memory/` 之外（仓库根目录、`knowledge-base/`、`.claude/skills/` 都不行）——那三处属于公开发布的内容；也不要在公开仓库里提交空目录或 `.gitkeep` 占位，目录一律运行时 `mkdir -p` 创建
8. ❌ 禁止只回一句"已保存记录"而不给实际路径；未落盘就明说"本次未落盘"

---

## 快速自检（每次更新后）

- [ ] profile.md 的技术栈/项目是否最新？
- [ ] current-state.md 的职业状态是否准确？
- [ ] career-timeline.md 是否有对应的时间线记录？
- [ ] 分析结果是否已归档到 analysis-archive/？
- [ ] 岗位池是否写在 `memory/job-pool/` 且覆盖前已把旧版放进 `history/`？
- [ ] `memory/` 之外（仓库根、`knowledge-base/`、`.claude/skills/`）有没有多出生产文件？
- [ ] 是否把实际写入的文件路径回报给用户了？
- [ ] 是否告诉用户更新了什么？
