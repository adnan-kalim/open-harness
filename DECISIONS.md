# Decisions

## 2026-11-04 20:32 — Build the harness as a measured experiment

Every improvement is a switchable feature that plugs into a fixed agent loop, and an eval runner compares profiles (`bare`, `full`, `full-minus-X`).
Why: the portfolio goal is to show that harness features improve a coding agent, not just that an agent can edit code. Plug-in features make each one a one-line toggle and keep the comparison fair.

## 2026-11-04 20:32 — TypeScript on Node 22, npm

Why: same ecosystem as Claude Code and MCP tooling, and Node and npm are already installed.

## 2026-11-04 20:32 — Multiple model providers from day one

Why: lets us test whether a feature helps every model, and whether a cheap model with the full harness can beat a stronger model without it. One OpenAI-compatible adapter covers most providers.

## 2026-11-04 20:32 — Free hosted models only, one strong and one weak

Why: no budget for API calls, and the laptop (4-core i7, 16 GB RAM, 2 GB GPU) is too weak for local models. Free-tier limits change often, so the two models get pinned at the start of step 1.

## 2026-11-04 20:32 — Two eval tiers: own small tasks, then SWE-bench Verified Mini

Why: our own tasks run fast on the laptop during development. SWE-bench Verified Mini (50 tasks, Docker inside WSL Ubuntu) gives credible headline numbers without needing a powerful machine. Download size to be confirmed before committing.

## 2026-11-04 20:32 — Compare `bare` against mini-SWE-agent

Why: shows the baseline was not made weak on purpose to make the features look better.
