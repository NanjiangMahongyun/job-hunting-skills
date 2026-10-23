---
name: memory-manager
description: 记忆库管理skill。负责读取、更新、归档用户的持久化记忆（profile/career-timeline/current-state + 按阶段归档目录 01-07），确保跨会话信息不丢失。每次对话开始时自动读取记忆库，重大变化时更新，分析结果归档。触发词：记住、更新档案、记录、归档、上次的情况、我之前的分析。
---

# 记忆库管理（跨会话持久化）

## 这个 skill 干什么

解决"过段时间回来找不到之前的交互记录"的问题。把用户的关键信息、分析结果、职业变化持久化到本地文件，下次对话直接读取接上。

**记忆库位置**：`memory/` 目录。**公开仓库里不放这个目录，也不放空目录占位**（git 不记录空目录，提交进去也没有效果）——目录由本 skill 在**第一次要写文件时**创建：`mkdir -p memory/01-resume memory/02-industry memory/03-job-pool/history memory/04-due-diligence/companies memory/04-due-diligence/rankings memory/05-negotiation memory/06-onjob memory/07-job-switch`；`rankings/[方向]/` 在第一次为该方向出排名时创建。

- `profile.md` — 用户基本档案（技术栈、项目、偏好、人际关系）
- `career-timeline.md` — 职业时间线（入职/离职/求职，按时间倒序）
- `current-state.md` — 当前状态快照（职业状态、求职进度、待办）
- `01-resume/` — 简历各版本（`resume-YYYY-MM.md`，旧版不删）
- `02-industry/` — 行业背景分析（append-only）
- `03-job-pool/` — 岗位池**当前工作集**（会反复增删，允许覆盖，但覆盖前必须把旧版另存进 `03-job-pool/history/`）
- `04-due-diligence/` — 公司尽调（append-only）
  - `companies/` — 公司尽调报告（跨方向共用）+ `尽调索引.md`
  - `rankings/` — 排名快照，按方向分子目录 + `排名索引.md`
- `05-negotiation/` — 谈判筹码 / 谈薪方案 / 合同审查（append-only）
- `06-onjob/` — 在岗记录（append-only）
- `07-job-switch/` — 跳槽决策 / 离职补偿（append-only）

**命名约定：目录用英文（01-resume、02-industry、03-job-pool、04-due-diligence、05-negotiation、06-onjob、07-job-switch，及其下固定子目录 companies/、rankings/），文件名用中文。** 目录是路径标识（会被相对链接、脚本引用），英文更稳；文件名给人看，中文信息量更高；数字前缀保证按求职阶段排序。按方向生成的一级子目录用简短英文/技术名（AI、Java、Python）；没有通用英文名的方向可保留中文，但同一方向始终用同一个名字。

**两类文件的区别是生命周期，不是主题**：尽调、排名、谈薪、行业分析、跳槽决策、在岗记录是"一次分析出一份结论"→ 进 `02/04/05/06/07` 对应阶段目录存档；岗位池是"随补搜不断改写的活文档"→ 进 `03-job-pool/`。把活文档塞进 append-only 的归档目录，只会逼人要么违反"不覆盖"、要么每次手抄一份。**尽调索引与排名索引是查表，不是分析结论**，允许覆盖；它们只指向最新报告/排名，历史文件仍只增不删。01—07 这套阶段划分与对外那张"下一步该做什么"的路线图（`knowledge-base/阶段行动导航.md`）一一对应——用户在导航里定位到自己处在哪一阶段，本 skill 就往对应的阶段目录落盘、并把 `current-state.md` 的所处阶段更新到同一格。

**公司报告与排名分开存的原因**：公司报告是"实体事实"，同一家公司换个岗位方向还能复用；排名是"某次筛选条件下的视图"，条件一变就不可比。混在一个目录里只靠文件名区分，方向多了会找不到；分开后按 `04-due-diligence/rankings/[方向]/` 浏览即可。

### 生产文件落点总表（本仓库唯一的落点权威，其他 skill 以此为准）

| 产出方 | 落点 | 文件名 | 覆盖规则 |
|---|---|---|---|
| company-due-diligence（单公司） | `memory/04-due-diligence/companies/` | `YYYY-MM-DD-公司尽调-[公司名].md` | 只新增；同日重跑加 `-2`；跨方向共用，复用前查有效期 |
| company-due-diligence（横向对比/总排名） | `memory/04-due-diligence/rankings/[方向]/` | `YYYY-MM-DD-[城市]-[关键条件].md` | 只新增；同日同条件重跑加 `-2`；文件头记录筛选条件与池版本 |
| company-due-diligence（尽调索引） | `memory/04-due-diligence/companies/` | `尽调索引.md` | 允许覆盖；只存公司→最新报告+评级+有效期，不存正文 |
| company-due-diligence（排名索引） | `memory/04-due-diligence/rankings/` | `排名索引.md` | 允许覆盖；只存排名→方向/条件/池版本，不存正文 |
| job-negotiation-strategist | `memory/05-negotiation/` | `YYYY-MM-DD-谈判筹码-[公司名].md` | 只新增 |
| salary-negotiation | `memory/05-negotiation/` | `YYYY-MM-DD-谈薪方案-[公司名].md` | 只新增 |
| contract-reviewer | `memory/05-negotiation/` | `YYYY-MM-DD-合同审查-[公司名].md` | 只新增 |
| severance-calculator | `memory/07-job-switch/` | `YYYY-MM-DD-离职补偿-[公司名].md` | 只新增 |
| industry-background-analyzer | `memory/02-industry/` | `YYYY-MM-DD-行业分析-[方向].md` | 只新增 |
| job-switch-decision | `memory/07-job-switch/` | `YYYY-MM-DD-跳槽决策-[方向].md` | 只新增 |
| onjob-management | `memory/06-onjob/` | `YYYY-MM-DD-在岗记录-[主题].md` | 只新增（轻量结论只改 `current-state.md`） |
| jd-analysis-and-resume-tailoring | `memory/01-resume/` ＋ `profile.md` 的"简历版本"字段 | `resume-YYYY-MM.md` | 新版本另存，旧简历不删 |
| job-recommender | `memory/03-job-pool/` | `[地区]-[方向].md`（如 `南京-AI方向.md`） | **允许覆盖**；覆盖前把旧版另存 `memory/03-job-pool/history/YYYY-MM-DD-[地区]-[方向].md` |
| skill-upgrade-maintainer | 直接改仓库内文件，不产数据文件 | — | 改动记进 git 提交信息 |

🚨 **不得落在这三处之外**：仓库根目录、`knowledge-base/`、`.claude/skills/`。这三处都属于要公开发布的内容，落进去就等于把真实公司名、真实薪资写进公开库。

**公开性边界（参考依据：`knowledge-base/公开发布与免责合规手册.md`）**：`memory/` 里是**真实的人、真实的公司、真实的薪资**。它们只留在本地——`/memory/` 已在 `.gitignore` 里，任何"把这包内容发到 GitHub/群里"的动作之前，先按该手册第四节（个人信息）与第九节（发布前自查命令）过一遍：真实主体一律换成虚构示例，而不是打码；他人提供的截图、聊天记录、offer、工资单属于他人个人信息与可能含商业秘密的材料，不对外发布。

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
4. 如果需要上次的分析细节，到 `memory/` 下对应阶段目录（02–07）找对应文件
5. 按第0步（前置）校验「求职身份」字段是否在有效期，过期或缺失则先向用户确认，确认结果再回写档案

### 场景2：用户提供新资料（如简历）

1. 提取关键信息（技能、经历、项目、目标）
2. 更新 `memory/profile.md` 对应字段
3. 如果是新简历，保存到 `memory/01-resume/resume-YYYY-MM.md`
4. 在 `memory/current-state.md` 更新"简历版本"字段
5. 告诉用户："已更新档案，记录了X条新信息"

### 场景3：完成一次重要分析（如公司尽调、谈薪策略）

1. 按落点总表保存：公司尽调进 `04-due-diligence/companies/`，排名进 `04-due-diligence/rankings/[方向]/`，谈薪/谈判/合同进 `05-negotiation/`，行业进 `02-industry/`，在岗进 `06-onjob/`，跳槽/补偿进 `07-job-switch/`
2. 若本次是公司尽调：每完成一家就更新 `memory/04-due-diligence/companies/尽调索引.md`；本批全部完成后生成排名文件，并更新 `memory/04-due-diligence/rankings/排名索引.md`
3. 在 `memory/career-timeline.md` 添加一条记录，关联归档文件
4. 在 `memory/current-state.md` 更新"最近一次分析"字段
5. 告诉用户："分析已归档到 memory/04-due-diligence/…（或对应阶段目录），下次可直接引用"

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

1. 目标文件：`memory/03-job-pool/[地区]-[方向].md`（当前工作集，同一地区＋方向全库只此一份在用的）
2. **要覆盖先留档**：把旧版原样复制为 `memory/03-job-pool/history/YYYY-MM-DD-[地区]-[方向].md`，再写新版
3. 新版顶部保留检索条件与口径（地区／方向／薪资底线／硬条件是否放宽），下次补搜不用重问
4. 在 `memory/current-state.md` 更新"岗位池：共X个（达标Y＋备选Z），最后刷新日期"
5. 回报："岗位池已更新到 memory/03-job-pool/xxx.md，旧版在 history/2026-10-21-xxx.md"
6. 🚨 岗位池**不进** append-only 的归档目录（02/04/05/06/07），也**绝不写到仓库根目录**——根目录是要公开发布的

---

## 归档文件命名规范

```
memory/
├── profile.md
├── career-timeline.md
├── current-state.md
├── 01-resume/                            ← 简历各版，旧版不删
│   └── resume-2026-10.md
├── 02-industry/                          ← 行业背景分析
│   └── 2026-10-21-行业分析-Java后端.md
├── 03-job-pool/                          ← 岗位池当前工作集
│   ├── 南京-AI方向.md                    ← 会反复增删，覆盖前旧版进 history/
│   └── history/
│       └── 2026-10-21-南京-AI方向.md     ← 被覆盖前的旧版
├── 04-due-diligence/                     ← 公司尽调
│   ├── companies/                        ← 公司尽调报告，跨方向共用
│   │   ├── 2026-10-21-公司尽调-XX科技.md
│   │   └── 尽调索引.md                   ← 公司 → 最新报告 + 公司评级 + 有效期，允许覆盖
│   └── rankings/                         ← 排名快照，按方向分目录
│       ├── AI/
│       │   └── 2026-10-21-北京-150以上-双休.md
│       ├── Java/
│       │   └── 2026-10-21-上海-180以上-双休.md
│       └── 排名索引.md                   ← 排名文件 → 方向/条件/池版本，允许覆盖
├── 05-negotiation/                       ← 谈判筹码 / 谈薪方案 / 合同审查
│   ├── 2026-10-21-谈判筹码-XX科技.md
│   ├── 2026-10-21-谈薪方案-XX科技.md
│   └── 2026-10-21-合同审查-XX科技.md
├── 06-onjob/                             ← 在岗记录
│   └── 2026-10-21-在岗记录-试用期融入.md
└── 07-job-switch/                        ← 跳槽决策 / 离职补偿
    ├── 2026-10-21-跳槽决策-Java后端.md
    └── 2026-10-21-离职补偿-XX科技.md
```

归档格式：`YYYY-MM-DD-[分析类型]-[对象].md`；岗位池格式：`[地区]-[方向].md`（不带日期，日期只在 `history/` 里）。
排名格式：`04-due-diligence/rankings/[方向]/YYYY-MM-DD-[城市]-[关键条件].md`；文件名只放日期+城市+关键差异条件，完整筛选条件写文件头，不塞进文件名。
同日重跑同一对象：文件名加 `-2`、`-3`，不覆盖旧报告。仅 `尽调索引.md` 与 `排名索引.md` 允许覆盖，且只存指向、不存正文。

---

## 与其他 skill 的配合

**每个产出分析结果的 skill 跑完都要调用本 skill 归档**，落点见上面「生产文件落点总表」，不要在各自的 SKILL.md 里另写一套路径：

```
company-due-diligence（单公司）  → 04-due-diligence/companies/YYYY-MM-DD-公司尽调-[公司名].md
company-due-diligence（排名）    → 04-due-diligence/rankings/[方向]/YYYY-MM-DD-[城市]-[关键条件].md
job-negotiation-strategist       → 05-negotiation/…-谈判筹码-[公司名].md
salary-negotiation               → 05-negotiation/…-谈薪方案-[公司名].md
contract-reviewer                → 05-negotiation/…-合同审查-[公司名].md
severance-calculator             → 07-job-switch/…-离职补偿-[公司名].md
industry-background-analyzer     → 02-industry/…-行业分析-[方向].md
job-switch-decision              → 07-job-switch/…-跳槽决策-[方向].md
onjob-management                 → 06-onjob/…-在岗记录-[主题].md（轻结论只更新 current-state.md）
job-recommender                  → 03-job-pool/[地区]-[方向].md（覆盖前旧版进 03-job-pool/history/）
jd-analysis-and-resume-tailoring → 01-resume/resume-YYYY-MM.md ＋ profile.md 的"简历版本"字段
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
- [ ] 分析结果是否已归档到对应阶段目录（02–07）？
- [ ] 公司尽调是否一家一份独立报告（`04-due-diligence/companies/`），并更新了 `尽调索引.md`？
- [ ] 横向对比/总排名是否放在 `04-due-diligence/rankings/[方向]/` 并更新了 `排名索引.md`？是否等本批范围内全部公司完成后才生成？
- [ ] 排名表是否分列公司风险与岗位风险，没有拿公司评级代替岗位评级？
- [ ] 岗位池是否写在 `memory/03-job-pool/` 且覆盖前已把旧版放进 `history/`？岗位池是否含投递链接核验表？
- [ ] `memory/` 之外（仓库根、`knowledge-base/`、`.claude/skills/`）有没有多出生产文件？
- [ ] 是否把实际写入的文件路径回报给用户了？
- [ ] 是否告诉用户更新了什么？
