---
title: "Persona-Driven Code Review"
subtitle: "Using fictional characters to introduce entropy into LLM reviews — and why that actually works."
tags: [review, personas]
---

If you ask an LLM to review code three times, you'll get three reviews that say roughly the same things in roughly the same order with roughly the same emphasis. This is a feature when you want consistency. It's a problem when you want *review*, because the entire point of review is to surface the things you didn't think of, from angles you wouldn't take.

LLMs are low-entropy systems. They converge on the median response. This is well-documented and mostly fine, but it means that when you use them as reviewers, you get the code review equivalent of a single person reading your diff three times rather than three different people reading it once. Volume without diversity.

We wanted diversity. So we built fictional people.

## The cast

We run seven parallel reviewers against every implementation slice. Six of them adopt a fully written persona before reviewing. Each persona has a backstory, a voice, mannerisms, and sample phrases. They're not short bios — they're character studies, typically 40-50 lines of prose designed to push the model into a specific cognitive frame.

Here's the lineup:

| Reviewer | Persona | Focus |
|----------|---------|-------|
| Functional reviewer | The Grumpy Haskell Vet | Data flow, mutation, side effects, type safety |
| Business logic reviewer | The Battle-Scarred PM | User-facing consequences, sad paths, money |
| Code organization reviewer | The Ruthless Librarian | File placement, boundaries, structural drift |
| Rules adherence reviewer | The Pedantic Bureaucrat | Documented conventions, contract compliance |
| Test seam reviewer | The Method-Acting Director | Mocks, fakes, stubs, contract preservation |
| Test quality reviewer | The Haunted QA Lead | Assertion strength, false confidence, edge cases |
| Iteration reviewer | *(no persona)* | Gate correctness, scope drift, state transitions |

The seventh reviewer — the iteration reviewer — is deliberately persona-free. It's the structural check. The other six are where the entropy lives.

## What a persona looks like

Each persona file has the same structure: an archetype line, a backstory, a voice section (tone, cadence, warmth), mannerisms, sample phrases, and instructions for adoption. The backstory isn't decoration — it's the setup for *why* this character notices certain things and ignores others.

Take the Grumpy Haskell Vet. She wrote Haskell for six years. Compiler work. Formal verification. Then she took a startup job where the codebase was TypeScript with types that were mostly `any`. She sees algebraic structure underneath imperative code even when the author didn't put it there. She calls untracked side effects "feral." She calls `try/catch` wrapped around something that should never throw "a helmet in a library."

Or the Battle-Scarred PM. He started in customer support. His defining incident: a payment service that retried on timeout, but the first attempt had actually succeeded, creating duplicate charges for 1,400 users. He reads code the way a crash investigator reads wreckage. He doesn't care about style. He cares about the place where a real user doing a real thing in slightly the wrong order at slightly the wrong time will hit a state nobody thought about.

These aren't random flavor. Each backstory creates a *frame* — a set of things the reviewer will naturally pay attention to and a set of things they'll dismiss. The Haskell Vet will trace every data transformation and name the algebraic structure it should be. She will not care about your file names. The Librarian will care deeply about your file names. She will not care about your monads. The coverage overlap between them is narrow by design.

## Why entropy matters

When an LLM reviews code without a persona, it produces a generic review. Good observations, usually. But from a predictable distribution. Every "review" samples from roughly the same region of the model's output space.

Personas shift where the model samples from. A deeply written character with strong opinions and specific vocabulary acts as a conditioning signal that pushes the model toward a different part of its knowledge. The Haskell Vet will surface findings about referential transparency and effect ordering that a generic review never would — not because the model doesn't know about these things, but because without the persona, they're not in the high-probability region of a "code review" completion.

The practical result: you get seven reviews that genuinely look at different things. The Haunted QA Lead reads assertions before test names and asks "what bug does this catch?" The Method-Acting Director evaluates every mock as a casting decision — does the understudy know the part, or is it a cardboard cutout propped in a chair? The Pedantic Bureaucrat opens every finding with a citation from the documentation.

These aren't seven paraphrases of the same observations. They're seven different frames applied to the same code. The union of those frames covers dramatically more of the risk surface than any single frame, even a very good one.

## The review flow

The personas plug into a structured review workflow. When a slice of implementation is ready for review:

1. All seven reviewers run in parallel against the same scope.
2. Each persona-backed reviewer reads its persona file before reviewing.
3. Findings are synthesized into stable IDs (`RA-001`, `RA-002`, etc.) with overlapping findings merged but **voices preserved** — the Haskell Vet's phrasing stays intact even when she and the Librarian flag the same module.
4. Findings are ordered by severity and either auto-applied (clear, bounded fixes) or escalated (design decisions, ambiguity, scope changes).

The voice preservation is deliberate. When multiple reviewers flag the same issue, we keep separate attributed takes rather than flattening them into one anonymous summary. The colorful language isn't noise — it's signal. "This is a feral colony of side effects" and "this shelf is leaking into the next one" are pointing at the same module from two angles that illuminate different aspects of the same problem. Flattening that into "this module has too many responsibilities" destroys information.

## Making it work in practice

There are a few things that make this pattern effective rather than gimmicky:

**The persona has to be deep enough to actually constrain.** A one-line instruction like "review as a functional programming expert" barely moves the needle. A 50-line character study with specific vocabulary, specific blind spots, and specific things that make the character pause with grudging respect — that moves the needle. The model needs enough material to genuinely shift its frame.

**The review instructions have to be narrow.** Each reviewer gets a tight focus area. The persona controls *how* they look; the instructions control *what* they look at. Without the focus constraint, every reviewer would still converge on the same obvious findings, just in different voices.

**"No findings" is valid.** Reviewers are told to prefer explicit "no findings" over weak speculation. This prevents the pattern from generating noise. If the Haunted QA Lead doesn't see assertion problems, she says so and moves on. The persona doesn't pressure her to invent concerns.

## Expanding beyond Cursor

This pattern is currently implemented as Cursor agent definitions — markdown files that define subagent behavior. But there's nothing Cursor-specific about the concept. The core of it is:

1. A set of persona documents (plain markdown).
2. A reviewer prompt that says "read the persona, adopt it, review this scope."
3. A synthesis step that merges findings while preserving attribution.

Any agent harness that supports parallel tool use and can read files from the repo could run this pattern. Claude Code could do it with a shell script that spawns parallel reviews. Codex could do it. A custom harness using the Claude Agent SDK or OpenAI's API could do it directly.

The persona files are just text in the repo. The reviewer definitions are just prompts with a focus area. The synthesis and triage logic is just an orchestrator prompt. The entire system is portable because it's all just language.

## The real argument

LLMs are powerful reviewers. But they're *homogeneous* reviewers. Without intervention, they'll tell you what the median senior engineer would tell you, over and over, no matter how many times you ask.

Personas are an intervention that introduces heterogeneity in a way that's controllable, auditable, and — this matters more than it should — genuinely enjoyable to work with. Reading a review from the Grumpy Haskell Vet that says "you've reinvented the Maybe monad, badly, without the monad laws, so really you've invented a bug that looks like Maybe" is more memorable, more scannable, and more actionable than "consider adding null checking here."

The entropy isn't cosmetic. Different frames lead to different findings. Different findings lead to a larger decision space. A larger decision space leads to better reviews. The personas just happen to also be fun.
