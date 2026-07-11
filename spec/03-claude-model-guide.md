# Which Claude model for which task (Pro plan)

Context: Claude Pro plan has a shared usage budget; heavier models burn it faster, roughly in
proportion to their API pricing — Haiku 4.5 (1×) → Sonnet 5 (~3×) → Opus 4.8 (~5×) →
Fable 5 (~10×) on input, similar ratios on output. Switch models per task in Claude Code
with `/model`.

## Per-task recommendations

| Task | Model | Why |
|---|---|---|
| Spec, architecture, "grill me" interviews | Fable 5 *(done — this session)* | Judgment-heavy, one-off; worth the spend once |
| Writing/iterating `evcc.yaml`, docker-compose, WireGuard walkthrough | **Sonnet 5** | Config work against good docs; Sonnet 5 is near-Opus on this at a third of the burn — the default workhorse for this whole project |
| Debugging device comms (Modbus timeouts, template quirks, log reading) | **Sonnet 5**, escalate to **Opus 4.8** if stuck after ~3 attempts | Most issues are config typos and network problems; escalate only for genuinely weird ones |
| Rabot tariff template in Go + upstream PR (Phase 2) | **Opus 4.8** (Sonnet 5 draft is fine) | Real code in an unfamiliar codebase + PR review standards; still doesn't need Fable |
| Runbook edits, translations, summarizing long logs, small doc fixes | **Haiku 4.5** | Fast and nearly free on the budget |
| Architecture pivots / multi-system mysteries (e.g. battery + tariff + charger interacting wrongly) | **Fable 5** | Reserve for problems where cheaper models have demonstrably failed |

## Practical rules

1. **Default to Sonnet 5.** For this project's remaining work (deploy, configure, verify) it
   is fully sufficient; don't pay the Opus/Fable premium for YAML.
2. **Escalate on evidence, not anxiety.** Two or three failed attempts on the same bug →
   one model tier up, bringing the full context (logs, config, what was tried).
3. **Batch small questions** into one message instead of many short turns — turns cost
   context re-reads.
4. **Give the phase doc as context.** Starting a work session with "read `spec/01-…` then do
   step X" gets better results per token than re-explaining the setup.
5. Deployment commands run on *your Pi*, not in Claude's sandbox — sessions produce
   configs/instructions; you paste and report back. Haiku/Sonnet handle that loop well.
