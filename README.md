# Stockkart Engineering Handbook

Cross-repository conventions for Stockkart: how the system fits together and the rules every change is held to.

| Document | Purpose |
|---|---|
| ARCHITECTURE.md | System context, components, data flow, boundaries, runtime layout |
| API_CONTRACT.md | How backend and frontend agree on interfaces, and how contracts change |
| REVIEW_RUBRIC.md | What reviewers check, severity definitions, definition of done |
| RUNBOOK.md | Running, testing and debugging locally; environment reference |
| decisions/ | Architecture decision records (ADRs): settled choices and how to propose changing one |

Per-repository rules live in each repository as AGENTS.md:
- https://github.com/Stockkart/inventory-api
- https://github.com/Stockkart/inventory-platform

Every document in this repository is given to the AI agents that design, implement and review changes, and reviewers must cite the rule they apply. Keep documents factual (derived from the code as it is) and short.
