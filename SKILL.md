---
name: section-aware-paper-notes
description: Use when a paper must be turned into section-aware structured notes with anchors and a paper card.
---

# 章节感知的结构化论文笔记

## 用途与适用条件

按章节切分论文，识别摘要/方法/结果/结论等角色，生成论文卡片、章节摘要、关键点、知识点与逐条定位。

适用条件：任务提供 `/input/case.json`（或 `--case` 指定的等价案例文件），按需提供 `/input/task.md` 和同目录的配套 JSON/文本/PDF/DOCX/XLSX 材料，要求把结构化、可复核的结果写入 `/app/output/result.json`。输入包含单篇论文正文或 PDF/DOCX/文本，要求生成可复核的结构化笔记而非泛泛摘要。

不适用：不用于多篇文献对照；不把参考文献条目当作研究结论；不生成没有原文依据的“启发”。

## 输入

- 主输入 `/input/case.json`：读取其中真实的段落、题录、字段表、工作单或查询；不根据文件名、标题或领域标签补造内容。
- 任务说明 `/input/task.md`：存在时优先作为任务语义依据；缺失时仅使用 `case.json` 中的显式字段。
- 配套材料：`comparison-workorder.json`、`target-fields.json`、`claims-to-verify.json`、`corpus-manifest.json`、`material.json`、`evaluation-workorder.json` 等按题目约定从 `/input` 与案例文件所在目录读取，文件名与解析结果写入输出的 `coverage`/`result`。
- 支持标准库可直接读取的 TXT/MD/CSV/TSV/JSON/BibTeX/RIS/XML/HTML，以及 DOCX/PPTX/XLSX（解包 XML）与 PDF（最小流解析）；扫描件或无文本层文件只登记状态，不伪装成已读。
- 路径解析顺序为 `/input`、案例文件所在目录、当前工作目录、`/app`；找不到的路径写入 `errors` 或 `uncertainty`，不静默替换。
- 仅依赖 CPython 3.9+ 标准库；不联网、不读取环境变量中的令牌、不修改 `/input`，只写 `--out` 与 `--work`。

## 处理流程

1. **读取论文**：取主文档，解析题录、正文、章节与页数；无文本层时如实降级。
2. **章节切分**：按标题行与编号切分，识别 abstract/introduction/method/results/discussion/conclusion/limitation/references 角色。
3. **论文卡片**：用研究问题、数据、方法、发现、局限五类模式句填充；缺失字段保持 `null`。
4. **章节笔记**：每节抽取前 3 个有效句子，生成 `summary` 与 `key_points`，跳过图表标题与过短句。
5. **知识点**：抽取定义句、缩写展开式与关键数值陈述，绑定段落定位。
6. **锚点校验**：每条笔记与卡片字段尽量给出页/行/节定位；找不到逐字匹配时标 `unverified`。
7. **契约落盘**：保留 `case_id` 与契约字段，章节不足时标 `partial`。

## 评分维度对应

- **结构完整**：论文卡片、章节、知识点与锚点四部分齐全。
- **章节忠实**：摘要不跨节拼接，参考文献不进入研究发现。
- **可追溯**：关键句带逐字引文与定位。
- **诚实降级**：字段缺失、章节不足、无文本层均明确说明。

## 输出契约

写入 `/app/output/result.json`（UTF-8 JSON，确定性排序，无随机、无时间戳）。顶层必须保留：

- 输入 `case_id`（缺失时为 `null`）；
- 输入 `output_contract` 声明的每一个字段名，原样保留；本 Skill 无法确定性映射的字段写 `null`，并在 `contract_notes` 说明原因，绝不编造；
- `status`：`success` / `partial` / `insufficient_input` / `failed`；
- `result`：完整结构化交付，避免契约字段映射丢失信息；
- `evidence`：可回到段落/文件/定位的证据列表；
- `uncertainty`：缺口、未读材料与不可判定项；
- `errors`：结构化错误或缺失路径；
- `stats`：计数、覆盖率与处理统计；
- `skill`、`challenge_id`。

主要交付段：`result.paper_card`、`result.sections`、`result.knowledge_points`、`result.anchors`、`result.coverage`。

字段名以实际 `output_contract` 为准；本 Skill 不删除、不改名、不放宽契约字段。

## 异常处理

| 情况 | 处理 | 输出表现 |
|---|---|---|
| `case.json` 缺失或不是合法 JSON | 不改写输入；写入结构化失败结果 | 退出码 `2`，`status` 为 `insufficient_input`（解析错误为 `failed`），`errors` 记录原因 |
| 配套工作单/字段表缺失 | 继续处理可用材料，不猜测清单 | `errors` 记录缺失文件，相关段为空并说明 |
| PDF 无文本层、文件损坏或无权限 | 登记文件状态，跳过该文件，继续其余材料 | `read_status`、`uncertainty` 明确标注 |
| 段落/字段/公式/指标找不到 | 保留空值，不推断 | 对应条目 `status` 为 `not_reported`/`insufficient_evidence`，`uncertainty` 说明 |
| 多个来源数值冲突 | 保留全部候选值与定位，不做平均或择一 | 标记 `source_conflict`/`conflict`，`evidence` 给出两处定位 |
| 输入只有单一文献却要求多文献比较 | 不伪造第二篇文献 | `status: partial`，明确说明无法构成多文档比较 |
| 运行异常 | 捕获并输出结构化失败，不假装成功 | `status: failed`，`errors` 含异常类型 |

## 限制

章节识别依赖标题格式，双栏 PDF 或扫描件可能顺序错乱；摘要是抽取式而非生成式，不补充原文没有的解释。

## 最小调用示例（示例，非官方案例结果）

```bash
python scripts/run_skill.py \
  --case /input/case.json \
  --out /app/output/result.json \
  --work /app/work
```

示意输入（合成，仅演示接口）：

```json
{"case_id":"synthetic-smoke","output_contract":{"notes":"object"},"documents":[{"title":"Synthetic Paper","text":"Abstract\nWe study X.\nMethods\nWe used Y.\nResults\nX improved by 10%."}]}
```

示意输出（合成，仅演示字段形状，不代表任何真实案例或评分）：

```json
{"case_id":"synthetic-smoke","paper_card":{"title":"Synthetic Paper","research_question":"We study X."},"sections":[{"title":"Abstract","role":"abstract","key_points":["We study X."]}],"status":"success","result":{},"evidence":[],"uncertainty":[],"errors":[],"stats":{},"contract_notes":{}}
```

## 自查清单

- [ ] `case_id` 与输入完全一致；`output_contract` 每个字段都存在，或明确记为 `null` 并写入 `contract_notes`。
- [ ] 每条结论/实体/字段都能回到 `evidence` 中的来源与段落定位；没有编造作者、年份、DOI、页码或引文。
- [ ] 未读、损坏、缺失、冲突与空结果都已登记，未写成“全部完成”。
- [ ] 重复运行产生相同结果；未联网、未修改 `/input`。


## 交付保证与评分对齐

调用顺序（固定）：

1. 先运行包内脚本（若存在）：`python scripts/run_skill.py`（默认读 `/input/case.json`，写 `/app/output/result.json`）；路径不同时用 `--case` / `--output` 指定。
2. 再完成脚本无法自动完成的实质工作（译文、论述、图表、修订、结论等），把结果写回 `result.json` 对应字段。
3. 最后自检：`case_id` 与输入一致；`output_contract` 每个字段都存在；`status` 使用本 Skill 输出契约的枚举（正常完成为 `success`，确实缺输入或存在未解决项为 `partial`，无法作答为 `insufficient_input`，运行异常为 `failed`）；无 `pending` 占位；数字、引用、来源可逐条回溯。

运行保证：

- 输入缺失、JSON 损坏、字段异常、输出目录不存在时，仍须写出结构化 `result.json` 并正常结束，不得崩溃或非零退出。
- 不联网、不伪造数据、不声称平台评分或官方案例结果。
- 交付物必须是可直接编辑与复核的成品，而不是流程说明。

评分对齐（本题五个维度）：

- 实际效果（40）：完整产出任务要求的最终成品，覆盖全部输入条目，可直接进入修改流程。
- 使用增益（25）：体现相对基础 Agent 的可验证改进（结构化、术语/数字/引用核对、可编辑性）。
- 创新增益（15）：保留方法学增量（分块、对齐、二次校验、版本比对等），并用结果证明其价值。
- 可靠体验（10）：异常输入有明确降级与说明，不同长度/文体/重复运行下稳定产出。
- 生态贡献（10）：用途、调用、输入输出、限制、示例、维护信息齐全，资源可被他人复用。

## 最终交付强制要求

1. **调用 Agent 必须交付成品，不是流程说明**：脚本产出只是索引、核对或规格骨架；调用 Agent 必须在同一 `result.json` 中补齐本题要求的最终产物（正文/译文/审稿意见/综述/分析结论/图表源文件与导出件/幻灯片页内容与讲稿/实验方案等）。交付字段为空、只有标题骨架，或含 `pending`、`TODO` 之类占位，一律视为未完成。
2. **必须覆盖全部输入条目**：输入中的每个章节、问题、文献、图、页面、指标都要有对应条目，不得只处理示例或首条；确实缺失的条目写入 `uncertainty` / `limitations` 并说明原因，不得静默跳过。
3. **必须自检后再交付**：`case_id` 与输入一致；`output_contract` 每个字段都存在；状态值取本题契约规定的枚举；数字、单位、引用、来源可逐条回溯；结果内部无自相矛盾。
4. **参数等价**：`python scripts/run_skill.py --case <输入> --out <输出>` 与 `--output <输出>` 等价；默认 `/input/case.json` → `/app/output/result.json`。
5. **平台用 3 个固定案例分别运行**：逻辑要由输入驱动，不针对某一份示例硬编码；字段命名差异走别名匹配与显式降级，并在 `uncertainty` 记录假设。
