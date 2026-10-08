# Relationship AI framework

The rules every AI feature in Mole follows: how Mole uses AI to solve memory,
what may be sent to a model provider (Gemini, Claude, OpenAI), how it's
minimised and pseudonymised, where it's stored, how Mole "learns" without
training on anyone's data, the layers and infrastructure, consent, quality,
safety and monitoring.

| File | What it is |
|---|---|
| [AI_FRAMEWORK.md](AI_FRAMEWORK.md) | **The framework.** Status: proposed. Decisions AI-1 to AI-30 in §12 are waiting to be agreed |
| [AI_FRAMEWORK_BRIEF.md](AI_FRAMEWORK_BRIEF.md) | The brief it answers: how AI worked in Mole on 5 Oct 2026, and the questions the framework had to settle |

The code it governs is in [mole-networking/Mole-V3](https://github.com/mole-networking/Mole-V3).
Mole-V3 refers to this repository for the framework and doesn't keep its own
copy, so changes to the framework are made here.
