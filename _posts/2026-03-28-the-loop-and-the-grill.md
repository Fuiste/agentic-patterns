---
title: "The Loop and the Grill"
subtitle: "Structured autonomy for coding agents, and why the first step is making them interrogate you."
tags: [workflow, prompting]
---

Anthropic published a [post](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) last year about long-running coding agents. The failure modes they describe are real: give a frontier model a big task, it tries to one-shot the whole thing, loses coherence halfway through, either leaves a half-finished mess or declares victory on something that isn't done.

Their fix was structured handoffs. Initializer agent decomposes the spec into features. Coding agent works on one feature per session, commits, leaves clean state. Progress files. JSON feature lists. Incremental, recoverable.

I'd been building the same thing and hitting the same walls. But I'd landed on two additions their post didn't cover: the agent should interrogate you *before* it decomposes anything, and the review step belongs inside the loop, not after it.

## The convergence problem

Tell an LLM to build something from a paragraph of description. It fills every gap with its best guess and starts building. Doesn't stop to ask. Doesn't flag assumptions. Resolves ambiguity immediately, silently, confidently.

This is the default. Every model I've worked with does this. Give it uncertainty and it collapses that uncertainty into a decision before you can intervene. The decision is always plausible. Often wrong. You won't find out until the assumption is load-bearing three layers deep.

The model *can* ask questions. It just won't, because asking questions looks like not making progress, and these things are trained to make progress. Resolution is output. Uncertainty is stalling. So they resolve.

There's something almost Baudrillardian about it — the model produces a simulation of understanding that's convincing enough to precede the real thing. The map before the territory. You get a confident decomposition of a spec the model never actually understood, and because it *looks* like understanding, everyone proceeds as though it is.

## The grill

Matt Pocock's [grill-me skill](https://www.aihero.dev/my-grill-me-skill-has-gone-viral) is the cleanest articulation of the fix. The whole skill is a few lines: *Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree resolving dependencies between decisions one by one.*

That's it. Works because it flips the objective. Instead of "produce output," the goal is "achieve understanding." The model stops resolving ambiguity and starts surfacing it. Forty-five minutes later you have answers to questions you didn't know existed.

What's actually happening: the grill forces you to confront the decisions you deferred. Every nontrivial project has a dozen of them. What happens when this fails? Is this idempotent? What's the error state? You were going to figure these out later. "Later" means the model guesses, and the model's guesses are always *reasonable* — never wild, never obviously wrong. Just the thing a competent developer would assume if they didn't ask. The reasonable guess is the most dangerous kind because you don't notice it's wrong.

The grill makes those decisions cheap. Up front, while nothing depends on them yet.

## Our version

We built this as the first phase of the workflow. Before implementation starts, a `/create-prd` step interrogates you. Structured choices where options are clear. Free-form questions where they're not. If major ambiguity remains after one round, it keeps going. Things that can't be answered get recorded as explicit assumptions and risks — not silently resolved.

Output is a PRD package: `prd.md` defining the work in gates (ordered phases, scoped deliverables) and `state.md` tracking progress. Designed to survive context loss. The next agent, next context window, whoever — should be able to read it cold and know what to build.

## The loop

Anthropic identified the right shape: decompose, then increment. But their architecture had review outside the loop. The coding agent implements, commits, moves on. Quality was "future work."

We put review inside the loop. Every slice gets reviewed before the next one starts.

1. **Implement one slice.** Bounded, reviewable in one pass. An `iterator` agent does this.
2. **Review the slice.** Seven [persona-driven reviewers](/persona-driven-code-review/) run in parallel.
3. **Triage.** Bounded fixes get auto-applied. Design decisions get escalated.
4. **Apply fixes.** Separate agent handles remediation. If behavior changes materially, one more review pass. Max two cycles per slice.
5. **Advance.** Update state. Identify next slice. Go to 1.

Loop terminates when: gate complete, blocker hit, escalations reach critical mass (3+ related), or a single slice fails review three times.

## Why inline review matters

Without it, assumptions accumulate. Slice 1 makes a reasonable choice about error handling. Slice 2 builds on it. Slice 3 builds on slice 2. By the time someone reviews, the choice from slice 1 is load-bearing across three layers. The reviewer can flag it. The fix is now a refactor.

With inline review, the wrong choice gets caught while it's still local. Blast radius: one slice. Same reason human teams review incrementally instead of at merge time.

Triage makes this practical. Not every finding needs a human. Missing null check, validation gap, weak assertion — auto-apply. Design decisions, scope questions, ambiguity — accumulate, present at gate boundary. The human reviews a batch of escalations, not a stream of interruptions.

## The state file

Every phase transition updates `state.md`. What was implemented, reviewed, auto-applied, escalated, and what comes next.

This is the answer to Anthropic's context window problem. New context window reads `state.md` and knows where things stand. No guessing, no git archaeology.

Every time the loop stops, it writes a concrete `next_action`. Not "continue working" but "Gate 1 complete. Review the PR. If approved, run `/loop-prd` to begin gate 2, which focuses on the state machine." Specific enough to act on without reading the conversation.

## The compound

These aren't independent patterns. They compound.

The grill produces a spec with explicit gates. The loop consumes it one gate at a time, one slice at a time, review between each. State file bridges context windows. Personas introduce review diversity. Triage keeps the loop moving.

Remove the grill, and the loop runs on hidden assumptions. The reviewers catch implementation bugs but not specification gaps. Remove the loop, and the grill produces a great spec that gets one-shot into a mediocre implementation. Remove inline review, and errors accumulate across slices.

The underlying observation: LLMs are convergent. They converge on the median interpretation of your spec, the median implementation of each feature, the median code review. Every part of this workflow is a mechanism to interrupt that convergence. Force the model to sit with ambiguity before resolving it. Force it to review from seven angles before advancing. Force it to stop after every slice instead of sprinting to done.

Not a slower agent. An agent that spends its tokens on the right things.
