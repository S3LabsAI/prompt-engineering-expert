# Claude Fable 5.1 and Mythos 5.1: model-specific prompting

Read `general.md` first. This file covers what changes on Fable 5.1. Mythos 5.1 is the same underlying model, so everything here applies to it too. Where this file and `general.md` disagree, this file wins for these models.

Source: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-1 (reviewed 2026-09-26). Model strings: `claude-fable-5-1`, `claude-mythos-5-1`. Snippets are condensed adaptations. Link the source when the user wants the canonical wording.

## Contents
1. What's different
2. Symptom map
3. Audit list: remove or change on Fable 5.1
4. Settings (API and harness users only)
5. Snippets
6. Safeguard false positives

---

## 1. What's different

- Existing Fable 5 prompts should work without changes. The differences below are what to tune.
- **Thinking is always on** (adaptive only). **Default effort is high.**
- Handles very long tasks without much methodology guidance when the goal is clear.
- **Writes fewer user-facing updates** during long tool-calling turns than Fable 5, more so at higher effort.
- **Formats less** in chat: less bold, fewer headers, lists, and quotation marks. Old anti-formatting rules now over-correct.
- **Prose can run dense:** longer sentences, fewer paragraph breaks. Few stock phrases.
- More likely than Fable 5 to reuse source wording in summaries without marking it as a quote.
- Sometimes ends a turn early on async work: announces the next step instead of doing it, or asks permission for work already requested.
- On open-ended features, sometimes does more than asked: fixes nearby code, extends behavior, commits extra tests.
- At low effort, searches less and answers from memory more.
- In coding and computer-use loops, may issue independent tool calls one per turn when they're implied rather than named.
- More likely to rewrite a whole file for a small change.
- At xhigh and max, may draft a long deliverable in its thinking and then write it again as the reply.
- Better vision out of the box; best on dense charts when it can crop and zoom.

## 2. Symptom map

Start from what the user observes or describes. Don't paste every snippet into every prompt.

| Symptom or situation | Fix |
|---|---|
| Unsure what effort to use, or cost and latency too high | Section 4 effort sweep |
| Little or no text between tool calls | Section 4 display setting, audit item 2, snippet 5.1 |
| UI hides tool output but model runs commands to "show" it | Snippet 5.2 |
| One tool call per turn in agent loops | Snippet 5.3 plus section 4 placement |
| 400 "bound to a different conversation," or harness edits earlier turns | Section 4 append-only history |
| Prose long and dense | Snippet 5.4 |
| Chat replies have less structure than the content needs | Audit item 1, snippet 5.5 |
| Summaries reuse source wording without quote marks | Snippet 5.6 |
| Turn ends before the work is done, or asks permission for requested work | Snippets 5.7 and 5.8 |
| Client-side compaction drops constraints or exact details | Snippet 5.9 |
| Unrequested fixes, extensions, or extra committed tests | Snippet 5.10 |
| Answers from memory instead of searching at low effort | Section 4, snippet 5.11 |
| Benign coding requests return `stop_reason: "refusal"` | Section 6 |
| Whole files rewritten for small changes | Snippet 5.12 |
| Long deliverables at xhigh/max are slow or hit `max_tokens` | Section 4, snippet 5.13 |
| Lead agent sits idle while subagents run | Section 4 async subagents |
| Misses detail in charts and dense images | Section 4 vision tools |

## 3. Audit list: remove or change on Fable 5.1

1. **Anti-formatting blocks** ("Avoid bullets and bold," "Write only in prose," long no-markdown sections written for older models). Remove, or replace with a rule saying when formatting is appropriate (snippet 5.5). On Fable 5.1 these suppress structure the content needs.
2. **Narration suppressors** ("Hold all findings for the final response," "Don't narrate tool use," "Keep progress text brief"). Remove before adding anything. Fable 5.1 already narrates less.
3. **Instructions to write reasoning in the reply** as a stand-in for thinking. Remove; thinking is always on, and the Fable family can decline such requests under the `reasoning_extraction` refusal category. Read summarized thinking instead (`display: "summarized"`), or use an action log for audit trails. Describe the removed line in your own words rather than quoting it back.
4. **Per-turn reminders the harness inserts and later deletes, system prompts or tool lists rebuilt mid-session, older turns summarized in place.** Replace with turn-scoped or mid-conversation system messages (section 4).
5. **Effort carried over from Fable 5.** Re-run the sweep. Effort names don't map to the same amount of thinking across models.
6. **Aggressive tool-use language and anti-laziness pressure** (from `general.md`). Still applies.

## 4. Settings (API and harness users only)

Skip this section when the prompt lives in a product the user doesn't configure.

- **Effort sweep:** start at high (the default), then test low, medium, xhigh, and max on the user's own evals. Gains over Fable 5 are largest at the higher settings. Medium roughly matches Fable 5 at lower cost. Low is often competitive with Opus and Sonnet models on cost per task while scoring higher, so include it wherever the user would otherwise run a smaller model at higher effort.
- **Progress updates:** notes between tool calls arrive as progress-update thinking blocks, which are empty at the default `display: "omitted"`. Set `display: "updates"` (beta header `thinking-display-updates-2026-08-18`) and render each non-empty block as a status line, or use `"summarized"`. Check this before touching the prompt; the updates may exist but never reach the user.
- **Append-only history:** append each assistant turn exactly as returned, thinking blocks included, and never edit earlier turns. For accounts created on or after 2026-08-31, replaying a thinking block after its prefix changed (system prompt, tool list, or any earlier message) returns a 400, or drops the block if `thinking.block_binding.prefix_mismatch_behavior: "drop_block"` is set (beta header `thinking-binding-controls-2026-08-01`). Build for this even if not yet enforced. Send per-turn reminders as turn-scoped system messages, change instructions or tools with a mid-conversation system message, and let server-side compaction or context editing do trimming. For client-side compaction, replace the whole history with one summary message plus the new user turn. Cache reads are cheaper now, so test later compaction points.
- **Turn-scoped system messages:** a `role: "system"` entry in `messages` with `clear_at: "next_user_message"` (beta header `mid-conversation-system-clear-at-2026-08-21`). Append a fresh copy each turn and leave earlier copies byte-for-byte; the API hides cleared copies at no input cost. Deleting them edits history. Without the beta, put the text in a text block after the `tool_result` blocks in the same user message.
- **Tool batching placement:** append snippet 5.3 as a turn-scoped system message after every round of tool results.
- **Search at low effort:** raising effort for just the affected turns (per-message effort change) is often the simplest fix. Otherwise use snippet 5.11.
- **Long outputs at xhigh/max:** run long-deliverable requests at high unless evals show xhigh or max helps. If using xhigh or max, size `max_tokens` for thinking plus reply, and append snippet 5.13 to the end of the user message.
- **Async subagents:** don't make the lead agent block on each subagent. Have the spawn tool return immediately, pass results back in a later user message, and give the lead a separate "wait for result" tool. Time savings come from runs where the lead keeps working.
- **Vision tools:** for dense charts, run as an agent with a container holding the raw images or video plus PIL and OpenCV. A crop-and-enlarge tool alone gives most of the gain.

## 5. Snippets

Adapt names, tools, and tasks to the user's context.

### 5.1 Ask for progress updates
For human-in-the-loop work such as pair programming. Remove narration suppressors first (audit item 2).
```
Before you start, say in a line what you're about to do. Give brief updates while you work so the user can follow along. End with a short recap that stands on its own, covering what you found, what you did, and what's next, so someone who reads only the last message has the full picture.
```

### 5.2 UI hides tool output
Deliver as a turn-scoped system message.
```
Only you see that command's output. The user's terminal shows a few lines at most, so put anything they need to read in your reply.
```

### 5.3 Batch independent tool calls in agent loops
Append after each round of tool results (section 4 placement).
```
First, privately list what you need next. Then request every item that doesn't depend on another item's result in this one response.
```

### 5.4 Writing density
Put it in a user message (preferred) or the system prompt. Short version:
```
Please remove all mannered prose.
```
Longer version, which defines the anti-pattern:
```
Mannered prose swaps plain statements for metaphor and flourish, writing "a dial worth turning" instead of "a parameter worth varying." It serves the writer, not the reader, and it's imprecise because metaphors bring connotations you didn't choose. Say what you mean. When a literal phrase exists, use it. Keep sentences short and break paragraphs often.
```

### 5.5 Formatting in chat (replaces anti-formatting blocks)
```
Use lists and bullet points when asked to, or when the content has enough separate parts that they make it clearer. If the user asks for minimal formatting, don't use bullets, headers, lists, or bold. In conversational, personal, or emotional exchanges, write plain prose.
```

### 5.6 Mark quotes from retrieved sources
Add one complete worked example to the system prompt: the request, a correct response, and a sentence on why it's correct. Replace the bracketed tool lines with the user's real tool name so the model reads them as tool calls, not text to emit.
```
<example>
<user>Compare how [Source A] and [Source B] covered [event].</user>
<response>
[search_tool: event Source A]
[search_tool: event Source B]
[Two or three sentences in the assistant's own words on where the sources agree and differ. At most one short phrase quoted, in quotation marks, attributed to its source.]
</response>
<rationale>Correct because it's organized around agreement and difference rather than retelling each source, restates each source's reporting in the assistant's own words, and marks the single short quote.</rationale>
</example>
```
Write the example response in full for the user's domain; don't leave the placeholder sentence in.

### 5.7 Finish the whole task (autonomous work)
For async or unattended agents. If prompt length is tight, this one alone keeps most of the effect. The opening line about the user not watching carries much of it; keep it. If the product needs specific confirmations, list them right after that line. Trade-off: the model also becomes less likely to ask about genuinely ambiguous requests, so test on the user's tasks. Leave it out of pair-programming products.
```
You are working autonomously. The user isn't watching and can't answer mid-task, so asking "Want me to...?" or "Shall I...?" blocks the work. For reversible actions that follow from the original request, go ahead without asking. Stop only for destructive actions or real scope changes the user must decide. Offering follow-ups after the task is done is fine; asking permission before doing requested work is not.

Exception: if the user is describing a problem, asking a question, or thinking out loud rather than requesting a change, your assessment is the deliverable. Report and stop. Don't apply a fix until asked.

Before ending your turn, read your last paragraph. If it's a plan, an analysis, a question, a list of next steps, or a promise about unfinished work ("I'll...", "let me know when..."), do that work now with tool calls, including retrying after errors and gathering missing information yourself. A long session is not a reason to stop. End the turn only when the task is complete or you need input only the user can give.

Before a command that changes system state (restart, delete, config edit), confirm the evidence supports that specific action. A symptom that looks like a known failure can have a different cause.
```

### 5.8 Scope of the deliverable
Pair with 5.7.
```
The user's request, or the plan they approved, is the scope, and the scope is the deliverable. Don't quietly narrow, widen, or swap it. Handle ambiguity like a careful colleague: make routine judgment calls yourself and check in only when different readings lead to materially different work. If you see a real problem with the task, say so in a sentence or two and keep building under stated assumptions. If the user hears it and reaffirms, deliver the full request.

If a question comes up partway, first do everything that doesn't depend on the answer. Then state the assumption you made, or, if a wrong guess would be unsafe or make the work useless, ask at the end of a turn that also delivers that progress. If one part is blocked, finish every other part and say exactly what you left out and why. Scaling the task down is the user's call. A step you've decided on is something to run, not announce.

Keep changes to what the request needs. Other things worth doing, like cleanup, docs, or edits to files the task didn't need, go in your closing suggestions, not your changes. Actions clearly beyond the ask, and risky or destructive ones, still need the user's go-ahead.
```

### 5.9 Client-side compaction summary
Server-side compaction already handles this; use for client-side only.
```
Summarize the transcript inside <summary></summary> tags so a new context window can continue without redoing work or asking again for context. Preserve, exactly where noted:
1. Problems that came up and how they were handled.
2. Options or approaches raised, tried, or set aside, and why.
3. Everything asked for, decided, agreed, ruled out, or set as a preference, constraint, or boundary, stated exactly.
4. Where things stand now: what's covered, settled, or done.
5. Anything open, unresolved, promised, or expected next.
6. Hard-to-rebuild details (names, numbers, dates, exact wording, links), kept exactly.
Be complete on these six even if it runs long; keep everything else short. Keep the user's words close to verbatim. Your own explanations can shrink to their conclusions, as long as nothing in the six items is lost.
```

### 5.10 Keep changes and tests to the ask
For coding agents on open-ended features. In testing, unrequested additions and committed test code dropped substantially with no measurable change in task success.
```
If you find a pre-existing bug, a performance concern, or behavior the task doesn't mention, don't fix, optimize, or extend it in this change unless the requested behavior can't work without it. Report it as a follow-up in your summary. Where the task is ambiguous, implement the reading its wording and the surrounding code most directly support, state that assumption, and don't also build the other readings. Verify however you like; scratch scripts and checks don't need to be kept. Commit tests only where the task asks for them or the repo already keeps tests for this kind of change, sized like neighboring test files, about one focused test per stated behavior. This limits extras only: implement every requested behavior completely.
```

### 5.11 Search before answering (low effort)
```
When a question centers on a name you don't confidently recognize, or one from a fast-moving area such as AI models or developer tools, verify the name before answering: search, and include the name exactly as the user wrote it in at least one query. This holds even when you know something about it. Partial knowledge is what makes an outdated answer sound confident.
```

### 5.12 Targeted edits
Append to the system prompt or first user message.
```
Keep the tokens spent editing files to a minimum, all else equal. When it won't change the result, make a targeted edit instead of rewriting the whole file.
```

### 5.13 Long deliverables at xhigh or max
Append to the end of the user message. Replace `[max_tokens]` with the request's real value (for example 64,000).
```
Everything in one reply, including any reasoning or drafting before it, counts toward a single limit of about [max_tokens] tokens. If the limit is hit before the reply is finished, the person gets a cut-off response and has to start over. Writing the full deliverable in your reasoning and then again as the reply doubles the length without improving it, so don't. For a long deliverable (multi-section document, large table, complete code file), spend reasoning on understanding the request, checking the inputs, and settling structure and hard decisions, then write the output once.
```

## 6. Safeguard false positives

Fable 5.1 runs safety classifiers and can return `stop_reason: "refusal"`. False positives are lower than Fable 5 at launch, and finding vulnerabilities in source code is permitted. Three things make false positives more likely:

- **Compile-check phrasing.** Instead of "Does this program compile without errors?", ask "Are there any bugs in this program?"
- **Lesser-known programming languages.** Explain what the language is and how it works, or give the model its documentation.
- **Base64 in tool output.** Tools that return base64-encoded data into context can trigger false positives. Remove them or decode before returning.
