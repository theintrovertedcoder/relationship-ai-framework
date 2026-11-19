# Relationship AI framework

The rules every AI feature in Mole follows: how Mole uses AI to solve memory,
what may be sent to a model provider (Gemini, Claude, OpenAI), how it's
minimised and pseudonymised, where it's stored, how Mole "learns" without
training on anyone's data, the layers and infrastructure, consent, quality,
safety and monitoring.

| File | What it is |
|---|---|
| [AI_FRAMEWORK.md](AI_FRAMEWORK.md) | **The framework.** Status: **agreed**, v2 (8 Oct); phases 0–3 built 29 Oct (awaiting deploy). Decisions AI-1 to AI-46 in §12 agreed on 8 Oct; AI-47 to AI-53 taken during the build, AI-54 on 19 Nov with the prompt-injection review (§10.7–§10.9). What every AI job costs, measured, is §6.5. What changed in v2 is listed at the top |
| [framework.html](framework.html) | The framework drawn out, as built on 29 Oct: the one door, mockups of the new screens, topology, layers, data classes, memory, speed and quality bars, and four features followed step by step with sample output. Open it in a browser. Live copy (v4, 29 Oct, as built): https://claude.ai/artifact/HsqJccHADH7WYLeenUv9ng |
| [docs/img/](docs/img) | The AI safety pictures (19 Nov 2026): [the six layers](docs/img/ai-safety-pipeline.svg) and [who writes the text each feature reads](docs/img/ai-safety-who-writes.svg). The prompt-injection review is §10.7–§10.9 |
| [AI_FRAMEWORK_BRIEF.md](AI_FRAMEWORK_BRIEF.md) | The brief it answers: how AI worked in Mole on 5 Oct 2026, and the questions the framework had to settle |

The code it governs is in [mole-networking/Mole-V3](https://github.com/mole-networking/Mole-V3).
Mole-V3 refers to this repository for the framework and doesn't keep its own
copy, so changes to the framework are made here.
