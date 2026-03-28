---
title: "Persona-Driven Code Review"
subtitle: "Fictional characters as entropy injection for LLM reviews."
tags: [review, personas]
---

Ask an LLM to review code three times. You get the same review three times. Different words, same distribution. Same emphasis, same ordering, same blind spots. The model converges on the median response because that's what it's built to do.

Fine for most things. Terrible for review. The whole point of review is to see what you didn't see. You need angles, not repetition.

So we built fictional people and made the model become them before reviewing.

## The cast

Seven reviewers run in parallel on every implementation slice. Six of them read a character study — 40-50 lines of backstory, voice, mannerisms, sample phrases — and adopt it fully before looking at code.

| Reviewer | Persona | Focus |
|----------|---------|-------|
| Functional | The Grumpy Haskell Vet | Data flow, mutation, side effects |
| Business logic | The Battle-Scarred PM | Sad paths, money, user-facing consequences |
| Code organization | The Ruthless Librarian | File placement, boundaries, drift |
| Rules adherence | The Pedantic Bureaucrat | Convention compliance, citations |
| Test seams | The Method-Acting Director | Mocks, fakes, contract preservation |
| Test quality | The Haunted QA Lead | Assertion strength, false confidence |
| Iteration | *(none)* | Scope drift, state transitions |

The seventh reviewer has no persona. Structural check. The other six are where the entropy lives.

## What the personas actually are

Not one-liners. Character studies. Each one follows the same shape: archetype, backstory, voice (tone, cadence, warmth), mannerisms, sample phrases, and adoption instructions.

The backstory does real work. It sets up *why* this person notices certain things and is blind to others.

The Grumpy Haskell Vet wrote Haskell for six years. Compiler work. Formal verification. Then she took a startup job where the types were mostly `any`. She sees algebraic structure underneath imperative code. Calls untracked side effects "feral." Calls `try/catch` on something that should never throw "a helmet in a library."

The Battle-Scarred PM started in customer support. His formative event: a payment service that retried on timeout, except the first attempt had already succeeded. Duplicate charges for 1,400 users. He reads code the way a crash investigator reads wreckage. Style doesn't register. He's looking for the place where a real user in a real hurry does the worst reasonable thing.

These backstories create frames. The Haskell Vet traces data transformations. She doesn't care about your file names. The Librarian cares deeply about your file names. She doesn't care about your monads. Narrow overlap by design.

## Why this works

Without a persona, the model produces a review from somewhere near the center of its distribution. Ask it to review from a specific, deeply characterized point of view and you shift where it samples from. The Haskell Vet surfaces findings about referential transparency and effect ordering that a generic review wouldn't — not because the model lacks that knowledge, but because it's not in the high-probability zone of a generic "code review" completion.

You're not getting seven paraphrases. You're getting seven genuinely different frames on the same code. The Haunted QA Lead reads assertions before test names. The Method-Acting Director evaluates every mock as a casting decision — does the understudy know the part or is it a cardboard cutout propped in a chair? The Bureaucrat opens every finding with a citation.

Seven frames. Narrow overlap. The union covers more risk surface than running any single reviewer seven times.

## The flow

When a slice is ready for review:

1. All seven reviewers run in parallel.
2. Each reads its persona file first.
3. Findings get synthesized into stable IDs — `RA-001`, `RA-002` — but voices stay intact. When the Haskell Vet and the Librarian flag the same module, you get both takes, attributed.
4. Findings get triaged: auto-apply (clear bounded fixes) or escalate (design decisions, ambiguity).

Voice preservation matters. "This is a feral colony of side effects" and "this shelf is leaking into the next one" describe the same module from two angles. Flattening both into "this module has too many responsibilities" destroys information.

## What makes it work vs. what makes it gimmicky

**Depth.** A one-liner like "review as a functional programming expert" barely moves the needle. 50 lines of specific vocabulary, specific blind spots, specific things that earn grudging respect — that moves the needle. The model needs enough material to actually shift frame.

**Narrow focus.** The persona controls *how* the reviewer looks. The instructions control *what* they look at. Without focus constraints, every reviewer converges on the same obvious findings. Just in funnier voices.

**Permission to say nothing.** Reviewers prefer explicit "no findings" over weak speculation. The persona doesn't pressure them to invent concerns. If the Haunted QA Lead doesn't see assertion problems, she says so and moves on.

## Portability

This is currently implemented as Cursor agent definitions. Markdown files. But there's nothing Cursor-specific about it. The whole thing is:

1. Persona documents. Plain markdown in the repo.
2. A reviewer prompt: read the persona, adopt it, review this scope.
3. A synthesis step that merges findings and preserves attribution.

Claude Code could run this. Codex could run this. Anything that can read files and run parallel completions could run this. The entire system is portable because it's all just text.

## The actual point

LLMs are homogeneous reviewers. Without intervention, you get the median senior engineer's take, over and over, regardless of how many times you ask. Personas break that homogeneity in a way that's controllable and auditable.

It also just turns out to be more fun. "You've reinvented the Maybe monad, badly, without the monad laws, so really you've invented a bug that looks like Maybe" sticks in your head. "Consider adding null checking here" doesn't. The memorable review is the one you actually act on.

Different frames, different findings, larger decision space, better reviews. The entropy isn't cosmetic.
