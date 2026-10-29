# Prompt Engineering Expert Skill

A Claude skill that writes, reviews, rewrites, and debugs prompts for Claude: system prompts, custom instructions, agent and harness prompts, CLAUDE.md files, and one-off task prompts. It tunes each rewrite to the model that will run it, because prompting advice is measured per model and a fix for one model can hurt another.

## How it works

`SKILL.md` walks through five steps:

1. **Identify the target model** from what the user names, a model string in the material, or where the prompt will run.
2. **Identify the prompt type** (chat system prompt, agent or harness prompt, task prompt, custom instructions) and whether the user controls API settings.
3. **Diagnose** reported symptoms, lines that hurt the target model, missing essentials, and risky domains.
4. **Rewrite** with the matching model guidance.
5. **Check and deliver** the optimized prompt, what changed and why, settings, and test inputs.

## Structure

```
prompt-engineering-expert/
├── SKILL.md                      # Skill instructions and workflow
├── CLAUDE.md                     # Copy of SKILL.md, so Claude Code uses the skill in this folder
└── references/
    ├── general.md                # Baseline best practices, per-model notes, audit list, migration checklist
    ├── opus-5-5.md               # Claude Opus 5.5 specifics
    ├── fable-5-1.md              # Claude Fable 5.1 / Mythos 5.1 specifics
    └── medical-guidance.md       # Verbatim per-model medication dosing blocks
```

The model files layer on top of `general.md` and win where they disagree. Sonnet 5.5, Sonnet 5, Haiku 5.5, Opus 5, Opus 4.8, and Fable 5 are covered by `general.md` section 10, which points to each model's docs page.

## Freshness

The guidance was reviewed against Anthropic's prompt engineering docs on 2026-10-28 (general and Opus 5.5) and 2026-09-26 (Fable 5.1). The review dates and source links are in the Freshness section of `SKILL.md` and at the top of each reference file.

## Installing

Copy `SKILL.md` and `references/` into a folder named `prompt-engineering-expert` in your skills directory, or upload the folder as a skill in Claude.
