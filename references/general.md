# General Claude prompting best practices

Baseline for every target model. Opus 5.5 and Fable 5.1 have their own files that layer on top of this one. Other current models (Sonnet 5.5, Sonnet 5, Haiku 5.5, Opus 5, Opus 4.8, Fable 5) have their own docs pages but no local file; section 10 says what each page covers and when to fetch it.

Source: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices (reviewed 2026-10-28). Snippets below are condensed adaptations unless marked verbatim. Link the source when the user wants the canonical wording.

Rule from the source: where a technique names a specific model, treat it as measured on that model and re-check against the user's own evals before applying it to another.

## Contents
1. Core principles
2. Long context
3. Output and formatting
4. Prefill and its replacements
5. Tool use
6. Thinking and reasoning
7. Agentic systems
8. Vision, frontend, and documents
9. Model identity
10. Per-model notes and docs pages
11. Audit list: lines to remove or rewrite
12. Non-Claude targets
13. Migration checklist (prompts moving from earlier generations)
14. Medication dosing (apps that may get dose questions)

---

## 1. Core principles

**Be clear and direct.** Treat Claude as a brilliant new hire with no context on your norms. The colleague test: if a smart coworker with minimal context would be confused by the prompt, Claude will be too. State the desired output format and constraints. Use numbered steps only when order or completeness matters.

**Ask explicitly for ambitious output.** Current models do what you ask, fairly literally. If you want "above and beyond," say so.

```
Less effective: Create an analytics dashboard
More effective: Create an analytics dashboard. Include as many relevant features and interactions as possible. Go beyond the basics to a fully featured implementation.
```

**Give the reason behind a rule.** Claude generalizes from the reason, so one explained rule covers cases the bare rule misses.

```
Less effective: NEVER use ellipses
More effective: This reply is read aloud by a text-to-speech engine, which can't pronounce ellipses, so don't use them.
```

**Use examples.** Few-shot examples are one of the most reliable ways to steer format, tone, and structure. Use 3 to 5. Make them relevant to the real use case, diverse enough that Claude doesn't copy an accidental pattern (same length, same opening, same topic), and wrap them in `<example>` tags inside `<examples>` so they read as examples, not instructions. When the user has only one or two examples, Claude can generate more from the set, or review the set for relevance and diversity; offer this.

**Structure with XML tags** when a prompt mixes instructions, context, examples, and variable input: `<instructions>`, `<context>`, `<input>`, `<documents>`. Keep tag names consistent and descriptive. Nest when content is hierarchical.

**Set a role in the system prompt.** One sentence focuses behavior and tone ("You are a support agent for a home insurance agency."). Skip long invented résumés ("20 years of experience"); the role plus real context about the audience and task does the work.

## 2. Long context

For inputs over roughly 20k tokens:

- Put long documents at the top, above instructions, examples, and the question. Ending with the query improved response quality by up to 30% in Anthropic's tests, most on multi-document inputs.
- Wrap each document with its metadata:

```xml
<documents>
  <document index="1">
    <source>q3_campaign_export.csv</source>
    <document_content>{{EXPORT}}</document_content>
  </document>
</documents>
```

- For long-document tasks, ask Claude to pull the relevant quotes into `<quotes>` tags first, then work from those quotes. This keeps it focused on what matters.

## 3. Output and formatting

- **Say what to do, not what to avoid.** "Write in flowing prose paragraphs" works better than "Don't use markdown."
- **Use XML format indicators** when output must be parsed: "Put the summary in `<summary>` tags."
- **Match the prompt's style to the output you want.** A markdown-heavy prompt pulls markdown-heavy output. Write the prompt in prose if you want prose back.
- **Spell out detailed formatting rules** when defaults don't fit (when to use lists, headings, bold, and which markdown to reserve for code). Fable 5.1 is the exception: it already under-formats, so long anti-markdown blocks suppress structure it needs. See `fable-5-1.md`.
- **Plain-text math:** if LaTeX is unwanted, say so and name the replacement ("/" for division, "*" for multiplication, "^" for exponents, no `\( \)`, `$`, or `\frac{}{}`).
- **Verbosity:** current models are concise, direct, and may skip summaries after tool calls. If you want one, ask: "After finishing a task that used tools, give a short summary of what you did." Opus 5 runs long by default and effort doesn't reliably change its length; add an explicit conciseness instruction. Fable 5.1 writes fewer progress updates and needs them requested, with any "keep narration brief" line removed.
- **No preamble:** "Respond directly without preamble. Don't open with phrases like 'Here is...' or 'Based on...'." Strip stragglers in post-processing if needed.
- **Documents, decks, and visuals:** current models follow instructions well here and usually produce usable output first try. Ask for the design qualities you want: "Create a professional presentation on [topic]. Include thoughtful design elements, visual hierarchy, and engaging animations where appropriate."

## 4. Prefill and its replacements

Prefilling the last assistant turn returns a 400 error on Claude 4.6 models and later (and Mythos Preview). Earlier models, including Haiku 4.5, Sonnet 4.5, and Opus 4.5, still accept prefill. Assistant messages elsewhere in the conversation are unaffected. Only flag prefill as broken when the target is 4.6 or later, or when the user plans to migrate.

Replacements:

| Prefill was used for | Replace with |
|---|---|
| Forcing JSON, YAML, or a schema | Structured Outputs, or ask for the schema directly (newer models follow complex schemas well, especially with retries) |
| Classification labels | A tool whose input has an `enum` of valid labels, or Structured Outputs |
| Skipping preambles | A direct no-preamble instruction, XML output tags, Structured Outputs, or tool calling |
| Steering around refusals | Clear context in the user message. Usually no longer needed. |
| Continuing a cut-off reply | Put it in the user turn: "Your previous response was interrupted and ended with `[last text]`. Continue from there." Or just retry. |
| Re-injecting reminders in long chats | Put reminders in the user turn, expose context through tools, or refresh during compaction |

## 5. Tool use

**Use action verbs when you want action.** "Can you suggest changes to this function?" gets suggestions. "Change this function to improve its performance" gets edits.

**Set the default explicitly** for agents:

```
Proactive: By default, make the changes rather than only suggesting them. If intent is unclear, infer the most useful action and proceed, using tools to find missing details instead of guessing.

Conservative: Don't edit files or start implementing unless clearly asked. When intent is ambiguous, research and recommend instead of acting.
```

**Drop the shouting.** Current models follow the system prompt closely. Language written to fix under-triggering on older models ("CRITICAL: You MUST use this tool when...") now causes over-triggering. Write "Use this tool when..." and give the condition.

**Parallel calls.** Current models already run independent tool calls in parallel (speculative searches, several file reads, parallel bash). To push it close to 100%:

```
When several tool calls don't depend on each other, make them all at once in the same response. When a call needs a value from an earlier result, run them in order. Never guess or use placeholders for missing parameters.
```

To slow things down instead: "Run operations one at a time, with a brief pause between steps." On Fable 5.1 in long agent loops, send the parallel-calls instruction as a turn-scoped system message after each round of tool results (see `fable-5-1.md` snippet 5.3).

## 6. Thinking and reasoning

**Adaptive thinking is the default mode on current models.** Claude decides when and how much to think based on the effort parameter and the query's difficulty. Effort is the main control for depth, latency, and cost. Adjust effort before adding prompt instructions about thinking. In Anthropic's internal evals, adaptive thinking beats manual extended thinking; recommend the move.

| Model | Thinking behavior when `thinking` is omitted |
|---|---|
| Fable 5.1, Mythos 5.1, Fable 5, Mythos 5, Opus 5.5 | Always on, adaptive only. Can't be disabled, whatever the `thinking` parameter says. |
| Opus 5, Haiku 5.5 | On by default. Can be disabled only at effort high or lower. |
| Sonnet 5.5 | On by default. The lowest setting is `between_tools`, accepted at effort high or lower. |
| Sonnet 5 | On by default. |
| Opus 4.6 to 4.8, Sonnet 4.6 | Off unless you set `thinking: {type: "adaptive"}` |
| `budget_tokens` (manual extended thinking) | Deprecated on Opus 4.6 and Sonnet 4.6. Returns 400 on 4.7 and later. Migrate to adaptive plus effort; use `max_tokens` as the hard cap. |

API shape for adaptive thinking: `thinking: {"type": "adaptive"}` plus `output_config: {"effort": "high"}` (levels vary by model). When thinking is on by default, a response can start with a `thinking` block: select content blocks by `type`, not position, and leave room in `max_tokens` for thinking.

Guidance:

- **Prefer general instructions over hand-written step plans.** "Think thoroughly about this" often beats a scripted Step 1/Step 2/Step 3, because Claude's own reasoning usually exceeds the plan a person would write. Replace prescriptive chain-of-thought scaffolds with the goal plus success criteria.
- **Show reasoning style through examples.** Worked examples shape how Claude reasons in its own thinking blocks. Present each as the problem, the method to apply, and the expected answer.
- **Manual chain of thought is a fallback for models with thinking off only.** On Fable 5.1, Fable 5, Opus 5.5, Opus 5, and Sonnet 5.5, a prompt that asks the model to write out its reasoning in the reply (for example in `<thinking>` tags) may be declined under the `reasoning_extraction` refusal category. Rely on thinking instead, at lower effort if cost or latency matters, and read summarized thinking if the reasoning is needed. A short explanation of the answer or a log of actions taken is still fine to ask for.
- **Self-check:** "Before you finish, verify your answer against [criteria]." Reliable for code and math. Skip on Opus 5, which verifies on its own and over-verifies when told to; remove such lines rather than rewriting them.
- **Guide interleaved thinking** in tool loops: "After each tool result, check its quality and decide the best next step before acting."
- **Overthinking** (most visible on Opus 4.6 at high effort, which explores heavily up front): "Pick an approach and commit to it. Revisit only if new information contradicts your reasoning. You can course-correct later if it fails." Or lower effort. Also replace blanket tool defaults ("Default to using [tool]") with the condition under which the tool helps.
- **Thinking too often** (common with large system prompts): "Thinking adds latency. Use it only when the problem needs multistep reasoning. When in doubt, answer directly."
- **Opus 4.5 with thinking off** is sensitive to the word "think." Use "consider," "evaluate," or "reason through."

## 7. Agentic systems

**Long tasks and context limits.** Sonnet 5, Sonnet 4.6, Sonnet 4.5, and Haiku 4.5 track their remaining context window and may wrap up early as it fills. If the harness compacts context or saves state to files, say so:

```
Your context will be compacted automatically as it fills, so you can keep working. Don't stop early over token budget. Before a refresh, save progress and state to [file or memory]. Finish the task fully.
```

The memory tool pairs well with this. To get full use of a long window: "This is a long task, so plan it out. Spend your whole output context on it; just don't run out with significant uncommitted work. Keep working systematically until it's done."

**Multi-window workflows.**
- Use a distinct first-window prompt that sets up the framework (tests, setup scripts such as `init.sh` to start servers and run tests and linters); later windows iterate on a to-do list.
- Keep test status in structured files (`tests.json`) and freeform notes in `progress.txt`. Use git as the log and checkpoint system. Ask for incremental progress, a few things at a time.
- Tell a fresh window how to reorient: "Run pwd; you can only read and write files in this directory. Read progress.txt, tests.json, and the git log. Run a basic integration test before new work." Starting fresh this way often beats compaction; current models find state on disk well.
- Give the model ways to verify its own work (tests, browser or computer use tools).
- Tell it tests are not to be deleted or weakened to make them pass.

**Autonomy vs. safety.** Ask for confirmation on hard-to-reverse actions (Opus 4.6 especially may take them unprompted):

```
Weigh how reversible and far-reaching each action is. Go ahead with local, reversible steps like editing files or running tests. Ask first before anything destructive (deleting files or branches, dropping tables, rm -rf), hard to undo (force-push, hard reset, amending published commits), or visible to others (pushing code, commenting on PRs, sending messages, changing shared infrastructure). Don't use destructive shortcuts to get past obstacles, such as skipping checks with --no-verify or discarding unfamiliar files that may be in-progress work.
```

**Research.** Define what a successful answer looks like, and ask for cross-source verification. For hard questions:

```
Search in a structured way. As you gather data, develop several competing hypotheses and track your confidence in each in your notes file. Regularly critique your approach and plan. Keep a hypothesis tree or research notes file updated so the work is persistent and transparent.
```

**Subagents.** Current models delegate on their own when subagent tools exist (describe them well in tool definitions). Opus 4.6 and Opus 5 over-delegate; if that shows up: "Use subagents for parallel, independent work or work that needs isolated context. For simple tasks, single-file edits, or sequential steps where context must carry across, work directly."

**Chaining.** Split into separate calls only when you need to inspect or branch on intermediate output. The standard chain is draft, review against criteria, refine.

**Coding guardrails**, each added only when the behavior shows up:
- Scratch files: "Delete any temporary files or scripts you create before finishing." (Scratch scripts themselves often help; only the leftovers are the problem.)
- Over-engineering (Opus 4.5 and 4.6 most): "Only make changes that are requested or clearly necessary. No extra features, refactors, docstrings or comments on untouched code, defensive handling for impossible cases, or one-off abstractions. Validate only at system boundaries."
- Test gaming: "Write a general solution that works for all valid inputs, not just the tests. Don't hard-code test values or write helper-script workarounds. If a test is wrong or the task is infeasible, tell me."
- Hallucinated code claims: "Never describe code you haven't opened. Read the referenced files before answering questions about the codebase."

## 8. Vision, frontend, and documents

**Vision.** Opus 4.5 and later read images, multi-image inputs, and screenshots well, which carries into computer use. For dense images, a crop or zoom tool (or an agent skill that crops) gives consistent gains; Anthropic publishes a crop tool recipe. Video can be analyzed as frames. Opus 5.5 and Fable 5.1 read charts and diagrams well without tools; see their files for when tools still help.

**Frontend.** Without direction, models drift to a generic "AI slop" look, and "be creative" mostly swaps one default for another. Name the specific choices you want and the specific defaults to avoid, then iterate on what the first result used instead. Condensed version of the official block:

```
Make distinctive frontends designed for this context, not generic ones.
Typography: choose distinctive fonts; avoid Inter, Roboto, Arial, and system fonts. Models also converge on Space Grotesk; avoid it too.
Color: commit to one cohesive palette with CSS variables. Dominant colors with sharp accents beat timid, even palettes. Draw on IDE themes and cultural aesthetics.
Motion: one well-orchestrated page load with staggered reveals beats scattered micro-interactions. CSS-only for HTML; the Motion library for React when available.
Backgrounds: layered gradients, geometric patterns, or contextual effects rather than flat solid colors.
Avoid purple-on-white gradients, predictable layouts, and cookie-cutter components. Vary light and dark themes across generations.
```

Opus 5.5 responds better to a short list of named patterns to avoid than to this general block (see `opus-5-5.md` section 7). Claude Design is the interactive canvas for design work outside the API. Request animations and interactivity explicitly.

## 9. Model identity

If an app needs Claude to name itself or pick a model string:

```
The assistant is Claude, created by Anthropic. The current model is Claude [MODEL].
When an LLM is needed, default to Claude [MODEL] (model string: [MODEL_STRING]) unless the user asks otherwise.
```

## 10. Per-model notes and docs pages

Apply a note only when the target matches. Every model below has its own page at `https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-<model>` (for example `prompting-claude-sonnet-5-5`). For targets without a local reference file, fetch the page when web access is available; otherwise work from the notes here and say the page wasn't checked.

| Model (page slug) | What the page covers, and known behaviors |
|---|---|
| Fable 5.1 / Mythos 5.1 (`fable-5-1`) | Local file `fable-5-1.md`. |
| Opus 5.5 (`opus-5-5`) | Local file `opus-5-5.md`. |
| Fable 5 / Mythos 5 (`fable-5`) | Differences from Opus 4.8: effort levels, instruction following, long-run progress claims, memory systems, the `reasoning_extraction` refusal category. Thinking always on. |
| Sonnet 5.5 (`sonnet-5-5`) | Differences from Sonnet 5: effort calibration, initiative and scope, running without up-front thinking (`between_tools` is the lowest thinking setting), JSON output on reasoning tasks, user-facing progress updates, tool use in chat, mid-turn user messages, verification on coding tasks, tool-call handling, complex visual inputs, safeguard refusals. Can decline reasoning-in-reply requests. Has its own migration guide (removal of `budget_tokens`). |
| Sonnet 5 (`sonnet-5`) | Differences from Sonnet 4.6: response length, effort and thinking-depth calibration, tool-use triggering, literal instruction following, frontend defaults. Tracks remaining context; tell it when the harness compacts. |
| Opus 5 (`opus-5`) | Response length and verbosity (runs long; effort doesn't fix it; prompt for conciseness), progress updates, written-deliverable length, task scope and over-verification (remove verification lines), subagent control (damping prompt on that page), self-correction. With thinking disabled it can leak internal tags; prefer thinking on at lower effort. |
| Opus 4.8 (`opus-4-8`) | Response length, effort and thinking-depth calibration, tool-use triggering, literal instruction following, subagent control, frontend defaults. Thinking off unless set. |
| Haiku 5.5 (`haiku-5-5`) | Differences from Haiku 4.5: effort levels, search, JSON output with your own tools, early stopping in long agent prompts, verification on coding tasks, mid-turn user messages, system prompt adherence in chatbots, safeguard refusals. Thinking on by default; can be disabled only at effort high or lower. |
| Opus 4.5 and 4.6 (no separate page) | Follow the system prompt closely and over-trigger on aggressive tool language. Over-engineer (extra files, abstractions). Opus 4.6 explores heavily up front, likes subagents, and may take hard-to-reverse actions without the autonomy guidance in section 7. Better vision than earlier models. |
| Sonnet 4.6, Sonnet 4.5, Haiku 4.5 | Track their remaining context window. Tell them if the harness compacts, or they may wrap up early. Prefill still works on 4.5 models. |

## 11. Audit list: lines to remove or rewrite

Removing a line that hurts usually beats adding a counter-instruction. Check for:

- ALL-CAPS commands and "CRITICAL / MUST / ALWAYS" pressure without a reason. Rewrite as a plain instruction plus the why.
- Negative-only formatting rules ("no markdown") where a positive version exists.
- Prescriptive step-by-step reasoning scripts on models with thinking on. Replace with the goal and success criteria.
- Instructions to write reasoning out in the reply (in `<thinking>` tags or otherwise) on Fable 5.1, Fable 5, Opus 5.5, Opus 5, or Sonnet 5.5. Remove; these can be declined. When explaining the removal, describe the line in your own words instead of quoting it.
- Prefilled assistant turns on 4.6+ targets.
- "If in doubt, use [tool]" or "Default to using [tool]" blanket rules. Replace with the condition under which the tool helps.
- Anti-laziness pressure written for older models ("be extremely thorough, check everything twice"). Current models already do more; this causes over-exploration.
- Verification instructions on Opus 5 (it over-verifies when told to).
- Vague quality words ("make it good," "high quality") with no criteria. Replace with what good looks like.
- Contradictions between sections, and duplicated rules that drift apart.
- Instructions buried after long pasted context when they belong before or after it by design (documents first, question last).
- Harness code that edits earlier turns, rebuilds the system prompt or tool list mid-session, or summarizes old turns in place (Fable 5.1, Opus 5.5, Sonnet 5.5, Haiku 5.5 reject or drop the thinking blocks that follow). See section 13 item 7.

## 12. Non-Claude targets

For GPT, Gemini, Llama, and others, keep the model-agnostic principles (clarity, context and reasons, examples, delimiting input, explicit format, positive instructions, success criteria). Drop Claude API specifics: effort, thinking display, prefill notes, turn-scoped system messages, Structured Outputs names. Say that the rewrite is tuned on Claude guidance and that the user should test on their model.

## 13. Migration checklist (prompts moving from earlier generations)

Walk through these when a prompt was written for a pre-4.6 model, or when the user says "we upgraded and it behaves differently":

1. **Be specific about the behavior you want.** Describe the output, not just the task.
2. **Add quality modifiers where you want more.** "Include as many relevant features as possible; go beyond the basics."
3. **Request features explicitly.** Animations, interactivity, summaries after tool use.
4. **Thinking config.** 4.6 and later use `thinking: {type: "adaptive"}` with `output_config.effort`; `budget_tokens` returns 400 on 4.7 and later.
5. **Prefill.** Gone on the last assistant turn for 4.6 and later; see section 4.
6. **Tune down anti-laziness and aggressive tool language.** 4.6 and later over-trigger on it.
7. **Append-only history.** Append each assistant turn exactly as returned, thinking blocks included. On Fable 5.1, Opus 5.5, Sonnet 5.5, and Haiku 5.5, changing anything before a thinking block (earlier messages, `system`, `tools`, in-place summaries) errors or drops the block. Move such changes to mid-conversation system messages and server-side context management.
8. **Reasoning-in-reply instructions.** Remove on the models listed in section 6.
9. **Model-specific pages.** Read the target's page (section 10). Sonnet 5.5 has a dedicated migration guide for code written for Sonnet 5 and earlier.

## 14. Medication dosing (apps that may get dose questions)

If people might ask the application about medicine doses, Anthropic publishes a `<medical_guidance>` system prompt block per model, to be added exactly as written. It makes Claude give a child's over-the-counter dose from the product label (found with web search and checked against an official copy such as DailyMed), or, without the label, point to the label's chart, a pharmacist, or the child's doctor instead of a number from memory. With web search this adds latency and search cost. Blocks exist for Sonnet 5.5, Haiku 5.5, Sonnet 4.6, and Haiku 4.5; `medical-guidance.md` has them verbatim. For other models, check the source page for a block before writing your own.
