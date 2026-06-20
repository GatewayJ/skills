# Codex Review Skills

这个仓库用于存放可复用的 Codex skills。每个 skill 独立放在 `skills/<skill-name>/` 下，后续可以继续新增多个 skill。

## 已有 Skills

- `code-review`：审查架构边界、架构形态、代码风格、对外暴露原则、注释、日志、单测质量、功能正确性、性能和一致性。
  - 支持轻量审查和严格审查两种模式。

## 目录结构

```text
skills/
  code-review/
    SKILL.md
    agents/openai.yaml
    references/review-checklist.md
    references/review-examples.md
```

新增 skill 时，在 `skills/` 下创建新目录，并提供独立的 `SKILL.md`。
