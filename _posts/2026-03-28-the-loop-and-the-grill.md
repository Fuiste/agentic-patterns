---
title: "The Loop and the Grill"
subtitle: "Structured autonomy for coding agents, and why the first step is making them interrogate you."
tags: [workflow, prompting]
---

Anthropic recently published a [post](https://www.anthropic.com/engineering/harness-design-long-running-apps) on harness design for long-running coding agents. The core architecture: a planner that expands a short prompt into a full spec, a generator that builds against it, and an evaluator that grades the output and sends feedback back to the generator. GAN-inspired. Separate the thing doing the work from the thing judging it.

One line from that post stuck with me: "When asked to evaluate work they've produced, agents tend to respond by confidently praising the work — even when, to a human observer, the quality is obviously mediocre."

That's the whole problem in one sentence. Models don't just converge on median output. They converge on median *self-assessment*. They produce something plausible, evaluate it as good, and move on. The feedback loop is broken because the critic and the artist are the same entity, and the entity is constitutionally incapable of being hard on itself.

Anthropic's fix was separation. Ours too. But we arrived at it from a different angle and ended up with two additions: the agent should interrogate you *before* it plans anything, and the review step belongs inside the implementation loop, not after it.

## The convergence problem

Tell a model to build something from a paragraph. It fills every gap with its best guess and starts building. Doesn't ask. Doesn't flag assumptions. Resolves ambiguity immediately, silently, confidently.

Every model does this. Give it uncertainty and it collapses that uncertainty into a decision before you can intervene. The decision is always plausible. Often wrong. You won't notice until the assumption is load-bearing.

The model *can* ask questions. It won't, because questions look like not-progress, and these things are optimized to produce progress. Resolution is output. Uncertainty is stalling. So they resolve.

There's something almost Baudrillardian about it — the model produces a simulation of understanding that's convincing enough to precede the real thing. The map before the territory. You get a confident spec decomposition from a model that never actually understood the spec, and because it *looks* like understanding, everyone proceeds as though it is.

Anthropic's planner agent does something interesting here: it takes a 1-4 sentence prompt and expands it into a full product spec, and they explicitly prompt it to "stay focused on product context and high-level technical design rather than detailed technical implementation." Their reasoning: if the planner specifies granular technical details upfront and gets something wrong, the errors cascade downstream. Better to constrain deliverables and let the agents figure out the path.

Smart. But still missing a step. The planner is still resolving ambiguity on its own. It's just doing it at a higher level of abstraction.

## The grill

Matt Pocock's [grill-me skill](https://www.aihero.dev/my-grill-me-skill-has-gone-viral) is the cleanest fix I've seen for this. The whole skill is a few lines: *Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree resolving dependencies between decisions one by one.*

Works because it flips the objective. Instead of "produce output," the goal is "achieve understanding." The model stops resolving ambiguity and starts surfacing it. Forty-five minutes later you have answers to questions you didn't know existed.

What's actually happening: the grill forces you to confront the decisions you deferred. Every nontrivial project has a dozen of them. What happens when this fails? Is this idempotent? What's the error state? You were going to figure these out later. "Later" means the model guesses, and the guesses are always *reasonable*. Never wild. Just the thing a competent developer would assume without asking. The reasonable guess is the most dangerous kind because you don't notice it's wrong.

We built this as the first phase. Before implementation, a `/create-prd` step interrogates you. Structured choices where options are clear. Free-form questions where they're not. If ambiguity remains after one round, it keeps going. Things that can't be answered get recorded as explicit assumptions and risks — not silently resolved.

Output is a PRD package: `prd.md` defining the work in gates (ordered phases, scoped deliverables) and `state.md` tracking progress. Designed to survive context loss. The next agent reads it cold and knows what to build.

## The loop

Anthropic's architecture has a planner, then a generator working in sprints, then an evaluator grading each sprint. The evaluator catches real gaps — features that are display-only, stubs that never got wired up, interactions that look right but break on use. They note that "tuning a standalone evaluator to be skeptical turns out to be far more tractable than making a generator critical of its own work." Same thing we found.

But their evaluator is one agent. One frame. One set of criteria. Better than self-evaluation, but still a single perspective.

We put review inside the loop and made it seven perspectives.

1. **Implement one slice.** Bounded, reviewable in one pass. An `iterator` agent does this.
2. **Review the slice.** Seven [persona-driven reviewers](/persona-driven-code-review/) run in parallel. Different frames, different blind spots.
3. **Triage.** Bounded fixes get auto-applied. Design decisions get escalated.
4. **Apply fixes.** Separate agent handles remediation. If behavior changes materially, one more review pass. Max two cycles per slice.
5. **Advance.** Update state. Identify next slice. Go to 1.

Loop terminates when: gate complete, blocker hit, escalations reach critical mass (3+ related), or a slice fails review three times.

The separation principle is the same as Anthropic's. Reviewers don't fix. Implementers don't self-approve. But instead of one evaluator with one set of grading criteria, we have seven reviewers with seven different frames. The [Grumpy Haskell Vet](/persona-driven-code-review/) traces data flow. The Battle-Scarred PM traces sad paths. The Haunted QA Lead reads assertions before test names. Narrow overlap. Wide coverage.

## Why inline

Without inline review, assumptions compound. Slice 1 makes a reasonable choice about error handling. Slice 2 builds on it. Slice 3 builds on that. By the time someone reviews, the choice from slice 1 is load-bearing across three layers. The reviewer flags it. The fix is a refactor.

With inline review, the wrong choice gets caught while it's local. Blast radius: one slice.

Anthropic saw this too. Their sprint contracts — where the generator and evaluator negotiate what "done" looks like before code gets written — are doing similar work. They're front-loading the evaluation criteria so the generator builds against agreed terms rather than its own assumptions. We're doing it differently (review after each slice rather than contract negotiation before), but the insight is the same: don't let the generator run unchecked.

Triage makes it practical. Not every finding needs a human. Missing null check, validation gap, weak assertion — auto-apply. Design decisions, scope questions, ambiguity — accumulate, present at gate boundary. The human reviews a batch, not a stream.

## The state file

Every phase transition updates `state.md`. What was implemented, reviewed, auto-applied, escalated, and what comes next.

Anthropic talks about context resets as essential — clearing the context window entirely and starting fresh with structured handoff state, because compaction alone doesn't solve context anxiety. Our state file serves the same function. New context window reads it and knows exactly where things stand. No guessing. No archaeology.

Every time the loop stops, it writes a concrete `next_action`. Not "continue working" but "Gate 1 complete. Review the PR. If approved, run `/loop-prd` to begin gate 2." Specific enough to act on without reading the conversation.

## The compound

One thing from Anthropic's post that resonated: "every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing." They demonstrated this by removing the sprint construct from their harness when Opus 4.6 made it unnecessary. The model got better, so the scaffolding that compensated for its weakness became dead weight.

That's the right frame. Every piece of our system — the grill, the gates, the inline review, the personas, the triage — is a bet that the model needs this specific constraint to produce good work. Some of those bets will expire as models improve. The sprint construct already did for Anthropic. Maybe inline review eventually becomes unnecessary. Maybe the grill becomes unnecessary when models learn to flag their own uncertainty.

But right now, in March 2026, all of these constraints are load-bearing. Remove the grill, and the loop runs on hidden assumptions. Remove the loop, and the grill produces a great spec that gets one-shot into mediocre implementation. Remove inline review, and errors compound across slices. Remove the personas, and you get the same review seven times.

The underlying observation: LLMs are convergent. They converge on the median interpretation of your spec, the median implementation, the median review, the median self-assessment. Every part of this workflow interrupts that convergence somewhere. Force the model to sit with ambiguity before resolving it. Force it to review from seven angles before advancing. Force it to stop after every slice instead of sprinting to done.

Not a slower agent. An agent that spends its tokens on the right things.
