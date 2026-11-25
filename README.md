# Runtime contract

Bohrium Playground Period 4 submission — challenge `product-arena-06-6d8cf72b` (track: `literature-reading`).

This repository holds the complete skill package for the challenge: the skill
definition, its executable pipeline and all supporting documents. No precomputed
answers, private data or credentials are included.

## Skill

- Name: `bohr-reading-structured-paper-notes`
- Summary: 当需要把真实论文整理成与章节对应、可检索、可回链原文的结构化笔记时使用。本题范围：文献总结与结构化笔记（章节对齐总结、论文卡片、知识点提取、锚点回链）。产出 result.json，包含运行时 output_contract 要求的全部字段，每条关键笔记附章节/段落/图表定位与原文短引，未报告和不确定内容如实写入 uncertainty。

## Contents

| File |
|------|
| `LICENSE` |
| `SKILL.md` |
| `assets/contract-aliases.json` |
| `assets/policy.json` |
| `assets/result-schema.md` |
| `environment/resources.yaml` |
| `environment/runtime.txt` |
| `examples/demo-case/agent_judgement.example.json` |
| `examples/demo-case/case.json` |
| `examples/demo-case/material.md` |
| `examples/demo-case/task.md` |
| `examples/quickstart.md` |
| `references/artifact-pipeline.md` |
| `references/contract-handling.md` |
| `references/evidence-discipline.md` |
| `references/failure-modes.md` |
| `scripts/artifacts.py` |
| `scripts/pptxgen.py` |
| `scripts/reading_ext.py` |
| `scripts/sk_base.py` |
| `scripts/sk_entry.py` |
| `scripts/sk_selftest.py` |
| `scripts/skill_profile.json` |
| `scripts/specialize.py` |
| `scripts/topic.py` |
| `README.md` |

## How to run

```bash
python scripts/sk_entry.py --input /input --work /app/work --out /app/output
```

A single deterministic pass reads `/input`, computes the deliverable and writes a
contract-complete `/app/output/result.json` plus the human-readable review files
described in `SKILL.md`. The same input always produces byte-identical output; no
network access is used, and fields the materials cannot support are reported
explicitly instead of being left empty.

## Notes

- Pure Python standard library; no network access required.
- Every claim is anchored to paragraph-level evidence; unknown items are reported
  in `uncertainty` instead of being fabricated.
