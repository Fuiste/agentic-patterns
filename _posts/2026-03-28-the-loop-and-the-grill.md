---
title: "The Loop and the Grill"
subtitle: "Why autonomous coding agents need structured loops — and why the best first step is making them interrogate you."
tags: [workflow, prompting]
---

Anthropic published a [blog post](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) last year about the failure modes of long-running coding agents. The core observation: when you give a frontier model a big task and a context window, it tries to one-shot the whole thing, loses coherence halfway through, and either leaves a half-finished mess for the next context window or declares victory on a project that isn't done.

Their solution was structured handoffs — an initializer agent that decomposes the spec into features, a coding agent that works on one feature per session and leaves clean artifacts for the next session. Progress files. Git commits. Feature lists in JSON because the model is less likely to corrupt JSON than Markdown. Incremental, verifiable, recoverable.

This resonated because I'd been building the same kind of system, independently, and hitting the same walls. But I'd arrived at two additional conclusions that their post didn't cover: first, that the agent should interrogate you *before* it decomposes anything, and second, that the review step should be inside the loop, not after it.

## The questioning problem

Here's what happens when you tell an LLM to build something from a paragraph of description: it fills in every gap with its best guess and starts building. It doesn't stop to ask you what you meant. It doesn't flag its assumptions. It resolves ambiguity silently, immediately, and confidently.

This is the default behavior of every model I've worked with. Give it uncertainty, and it will collapse that uncertainty into a decision faster than you can blink. Sometimes the decision is good. Often it's plausible but wrong in ways you won't discover until the implementation is three layers deep and the assumption is load-bearing.

The problem isn't that the model can't ask questions. It's that asking questions feels like not making progress, and models are relentlessly trained to make progress. Resolution looks like output. Uncertainty looks like stalling. So they resolve.

Matt Pocock's [grill-me skill](https://www.aihero.dev/my-grill-me-skill-has-gone-viral) nailed this insight in a beautifully minimal way. The entire skill is a few lines: "Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree resolving dependencies between decisions one by one."

That's it. And it works extraordinarily well because it reframes the model's objective. Instead of "produce output," the objective becomes "achieve understanding." The model stops trying to resolve ambiguity and starts *surfacing* it. Forty-five minutes of grilling later, you have a conversation full of answers to questions you didn't know you needed to answer.

## What questioning actually does

When a model grills you on a spec, something specific happens: it forces you to confront the decisions you deferred. Every non-trivial project has a dozen of these. "What happens when this fails?" "Do you need this to be idempotent?" "Is this user-facing or internal?" "What does the error state look like?"

You know these questions exist. You were going to figure them out later. But "later" means the model guesses, and the model's guesses have a specific failure mode: they're always *reasonable*. They're never wild. They're the thing a competent developer would assume if they didn't ask. And because they're reasonable, you don't notice they're wrong until the gap matters.

The grill forces those decisions up front, while they're cheap to make and cheap to change. The resulting spec isn't just more detailed — it has fewer hidden assumptions, which means the implementation has fewer places where a plausible-but-wrong guess is silently load-bearing.

We built this into our workflow as the first phase. Before any implementation begins, a `/create-prd` skill interrogates the user about the initiative. It uses the `AskQuestion` tool to present structured choices where options are clear, and free-form questions where they're not. If major ambiguity remains after one round of clarification, it keeps going. Things that can't be answered get recorded as explicit assumptions, unknowns, or risks — not silently resolved.

The output is a PRD package: a `prd.md` that defines the work in gates (ordered phases with clear scope and deliverables), and a `state.md` that tracks progress. The PRD is designed to survive context loss — the next agent, or the next context window, should be able to pick it up cold and know what to build.

## The loop

Anthropic's post identified the right shape: decompose the work, then make incremental progress, one piece at a time, leaving clean state for the next session. But their architecture had the review step outside the loop. The coding agent implements, commits, and moves on. Quality assurance was mentioned as "future work."

We put the review inside the loop. Every slice of implementation gets reviewed before the next slice begins. The loop looks like this:

1. **Implement one slice.** Small, bounded, reviewable in one pass. An `iterator` agent does this work.
2. **Review the slice.** Seven [persona-driven reviewers](/persona-driven-code-review/) run in parallel, each looking at the slice from a different angle.
3. **Triage findings.** Clear, bounded fixes (bugs, missed validations, contract violations) get auto-applied. Design decisions and ambiguity get escalated for human review later.
4. **Apply fixes.** A separate `feedback-implementer` agent handles remediation. If the fix materially changes behavior, one additional review pass is allowed. Max two review cycles per slice to prevent infinite loops.
5. **Advance.** Update the state file with what was implemented, reviewed, applied, and escalated. Identify the next slice. Go to step 1.

The loop terminates on four conditions: the gate is complete, a blocker is hit, escalated findings reach critical mass (three or more related findings that could affect remaining work), or a single slice fails review three times (something structural is wrong).

## Why the loop matters

The key insight from Anthropic's post was that agents fail when they try to do too much at once. Our addition is that agents also fail when they defer quality checks to the end.

Without inline review, the implementation accumulates assumptions. Slice 1 makes a reasonable choice about error handling. Slice 2 builds on that choice. Slice 3 builds on slice 2. By the time a reviewer looks at the result, the choice from slice 1 is load-bearing across three layers and the cost of changing it is high. The reviewer might flag it, but the fix is now a refactor, not a tweak.

With inline review, the wrong choice gets caught while it's still local. The blast radius of a bad decision is one slice, not three. This is the same reason human teams do incremental code review rather than reviewing the entire feature at merge time — small diffs are cheaper to reason about and cheaper to fix.

The triage system makes this practical rather than tedious. Not every finding needs a human. A missing null check, a validation gap, a test that asserts the wrong thing — these can be auto-applied without stopping the loop. Design decisions, scope questions, and ambiguous trade-offs get accumulated and presented at the gate boundary. The human reviews a coherent batch of escalations rather than being interrupted after every slice.

## The state file

The mechanism that ties this together is the state file. Every phase transition updates `state.md` in the repo. It tracks what was implemented, what was reviewed, what was auto-applied, what was escalated, and what should happen next.

This directly addresses Anthropic's observation about context window handoffs. When a new context window picks up the work, it reads `state.md` and knows exactly where things stand. No guessing, no git archaeology, no "let me look around and figure out what happened." The state file is the progress file, the feature list, and the session handoff rolled into one.

Every time the loop stops — for any reason — it writes a concrete `next_action` into the state file. Not "continue working" but "Gate 1 is complete. Review the PR. If approved, run `/loop-prd` to begin gate 2, which focuses on the state machine." Specific enough that the human (or the next agent) can act on it without reading the conversation.

## The compound effect

The questioning phase and the loop aren't independent patterns. They compound.

The grill produces a spec with explicit gates, each with clear scope and deliverables. The loop consumes that spec one gate at a time, one slice at a time, with review between each slice. The state file bridges context windows. The personas introduce review diversity. The triage system keeps the loop moving without sacrificing quality.

Remove the questioning, and the loop runs on a spec full of hidden assumptions. The reviewer agents catch some of them, but they're catching implementation bugs, not specification gaps. Remove the loop, and the questioning produces a great spec that gets one-shot into a mediocre implementation. Remove the inline review, and the loop produces incremental work that accumulates errors across slices.

The whole system is designed around one observation: LLMs are relentlessly convergent. They converge on the median interpretation of your spec. They converge on the median implementation of each feature. They converge on the median code review. Every intervention in this workflow — the grilling, the loop, the personas, the triage — is an attempt to slow down or redirect that convergence. Make the model sit with ambiguity before resolving it. Make it review its own work from seven angles before advancing. Make it stop after every slice instead of sprinting to the end.

The result isn't a slower agent. It's an agent that spends its tokens on the right things.
