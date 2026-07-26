---
layout: default
title: "Inside Storykept's StoryEngine: turning a conversation into a family story"
header_title: "Storykept's StoryEngine"
date: 2026-07-26
description: "How Storykept transcribes, interviews, structures, rewrites, merges, and fidelity-checks a family memory without losing the original."
author: George Kostov
permalink: /posts/storykept-story-engine/
---

# Inside Storykept's StoryEngine: turning a conversation into a family story

<div class="article-lead">
  <p>A memoir app should not behave like a dictation tool with a “make this prettier” button. It has to listen for the shape of a memory, ask what is missing, preserve who said what, and improve the prose without quietly improving the facts.</p>
</div>

<figure class="article-figure article-figure--hero">
  <img src="/assets/images/projects/storykept-capture.png" alt="Storykept voice capture screen showing a family story being recorded">
</figure>

I'm building [Storykept](https://storykept.app), a private, voice-first place for families to record their stories. The first real user is my father, which makes Bulgarian support and a low-friction recording experience product requirements rather than localization polish.

Behind the recording screen is a set of typed AI capabilities we call the **StoryEngine**. It is not one giant prompt and it is not tied to one provider. It is a pipeline with explicit contracts for transcription, interviewing, rewriting, merging, composing, narration, and fidelity checking.

The architectural rule is simple: **the recording and raw transcript are evidence; everything the AI produces is a derived version.** We keep those layers separate.

<figure class="article-figure">
  <img src="/assets/images/storykept/story-engine.svg" alt="StoryEngine pipeline from audio and conversation through transcription, analysis, synthesis, fidelity checking, and family-facing story outputs">
  <figcaption>The same engine supports solo recording, invited contributors, typed stories, and live family interviews.</figcaption>
</figure>

## Start with capabilities, not model names

The public interface of the engine looks more like a set of small services than an “AI module”:

```ts
Transcriber.transcribe(input)
Analyzer.analyze(input)
Rewriter.rewrite(input)
Merger.merge(input)
Composer.compose(input)
Synthesizer.synthesize(input)
```

Each capability owns a different failure mode.

- The **transcriber** must preserve the speaker's words, language, timing, and corrections.
- The **analyzer** must understand the story so far and ask one useful next question.
- The **rewriter** turns a single person's raw transcript into readable first-person prose.
- The **merger** combines several labeled contributions without blending voices or dropping facts.
- The **composer** converts an interleaved live interview into the subject's story, keeping the interviewer's questions out of the narrator's voice.
- The **synthesizer** creates optional neutral narration. It never clones the storyteller's voice.

The application asks for a capability. A central model catalog decides which provider and model currently implement it. That keeps provider SDK calls out of capture routes and lets us change a model after an evaluation without rewriting the product flow.

Today, for example, the hot-path analyzer uses Claude Haiku 4.5 with `gpt-4o-mini` as a cross-provider fallback. Final rewriting, merging, composition, and fidelity checking use a stronger-model chain. Bulgarian transcription has its own language route. These are routing decisions, not business logic.

## The analyzer is the interviewer

After each recorded answer, the analyzer receives the recent conversation plus compact state accumulated from earlier turns. It returns strict structured data:

```json
{
  "isComplete": false,
  "completenessAnalysis": {
    "setting": "complete",
    "people": "complete",
    "setup": "partial",
    "action": "missing",
    "outcome": "missing",
    "significance": "partial"
  },
  "nextQuestion": "What is one morning in the workshop you can still picture clearly?",
  "entities": {
    "people": ["Grandfather Stefan"],
    "places": ["the radio workshop"],
    "dates": ["late 1960s"],
    "themes": ["craft", "patience", "family work"]
  },
  "templateGuess": "character_portrait",
  "templateConfidence": "high",
  "summary": "The storyteller remembers visiting Grandfather Stefan's radio workshop in the late 1960s and learning patience by watching him repair sets."
}
```

That is a synthetic example, but the shape is the production contract.

The analyzer tracks six completeness axes and classifies the memory into one of nine narrative templates: anecdote, life event, character portrait, period, turning point, relationship, origin, ritual, or reflective case study. The template changes what “missing” means. A character portrait needs revealing scenes and traits; it should not be forced through the chronology of a life event.

The classification also changes the next-question ladder. Instead of endlessly asking “what happened next?”, the engine can ask for a sensory detail, a moment that proves a trait, the consequence of a decision, or why the memory still matters.

There are several less visible safeguards:

- Known people, places, and dates are carried forward so an early detail is not forgotten.
- Questions already shown are passed back as an avoid list.
- Explicitly rejected tags do not return.
- Spoken corrections such as “Pernik—no, Kremikovtsi” produce a `correctedAway` value that remains suppressed on later turns.
- The family roster is expressed from the current narrator's perspective, so “my daughter” and a known person's name can resolve to the same human.

This is where a generic chat completion becomes an interviewing system: memory is not just context; it is typed, persisted state with rules.

## Long interviews need a memory boundary

Our first implementation resent the full transcript on every turn. It was easy to reason about and expensive in the one currency that mattered: waiting.

In a 30-day production snapshot, the analyzer's median response was four seconds, p95 was 10.3 seconds, and the slowest call took 26.7 seconds. The output averaged only 172 tokens. The input—5,185 tokens on average—was the problem.

The current path keeps a faithful running summary, completeness grades, entity sets, prior questions, and turn count. Once an interview becomes long, the model sees that established state plus the last three exchanges. The static instruction prefix is cached.

We did not turn this on because it looked architecturally elegant. We compared full-history and windowed runs on long English and Bulgarian fixtures. The two paths returned identical completeness, identical completion decisions, byte-identical next questions, and full entity retention. The feature remained behind a kill switch until that evaluation passed.

That sequence matters: compressing conversational history is easy. Proving that the compression did not erase somebody's grandmother's name is the real work.

## Finishing is a different job from interviewing

When the storyteller taps finish, the engine stops asking questions and selects a synthesis path.

For a solo recording or typed contribution, the rewriter receives the full raw transcript, the detected story type, canonical people and place names, and a treatment:

- **As told** removes disfluencies and repairs readability while staying close to the speaker's words and order.
- **Literary** applies more scene, rhythm, and narrative craft without permission to invent facts.

Both return the same family-facing shape:

```json
{
  "title": "The Radio on the Workbench",
  "previewSummary": "A childhood memory of quiet mornings spent watching a grandfather repair radios by hand.",
  "polishedNarrative": "Grandfather Stefan never hurried a radio. On Saturday mornings I would sit beside his workbench and watch him test one connection after another...",
  "costUsd": 0.0042
}
```

Again, the text is synthetic; the interface is real.

If several relatives contribute, the merger receives ordered blocks with explicit contributor labels. Its job is not to average them into one omniscient narrator. It must retain attribution, preserve disagreement, and keep every contribution represented. A separate live-call composer handles interviewer/subject conversations and writes only from the subject's first-person perspective.

Long stories are sectioned so a one-hour interview does not get silently truncated by a single model call. Sections are processed and recombined under “keep every fact” checks rather than treated as unrelated summaries.

## Polish needs an adversary

A good rewrite can be more dangerous than a bad one. Fluent prose makes a changed year, flipped relationship, or invented cause feel authoritative.

StoryEngine therefore has a separate fidelity role. It compares the polished story with the raw transcript and breaks the prose into atomic factual claims. Each claim is labeled:

- `grounded`: the transcript supports it
- `inferred`: plausible, but not actually stated
- `unsupported`: absent from or contradicted by the transcript

It also grades importance. A harmless atmospheric phrase is different from a changed name, date, place, relationship, number, cause, or outcome. Only consequential weakly grounded claims cross the flagging threshold.

The checker specifically hunts for the failures fluent models hide well: dropped negation, sarcasm taken literally, uncertainty hardened into a fact, a detail assigned to the wrong relative, or an interviewer's comment placed in the storyteller's mouth.

The report runs after finish and does not destroy the user's result if a verifier is temporarily unavailable. It gives us a quality baseline and a path to ask a warm correction question when a consequential claim cannot be grounded.

## Every AI call leaves a trail

Model output is operational data, so every capability logs the same core fields:

- task and implementation version
- story and contribution attribution
- language
- latency
- input and output token counts
- cost
- hashed input identity
- structured output or error

Logging is scheduled after the response on interactive routes, so the audit insert does not add another database round trip to the storyteller's wait.

Those records support cost accounting, failure alerts, model comparisons, and replayable evaluation. More importantly, they stop model choices from becoming folklore. When the analyzer feels slower, we can query the tail. When a prompt changes, we can put the exact production builder into the benchmark.

## The output is a small family archive, not an AI artifact

The family eventually sees a title, preview, readable story, people/place/theme tags, the original audio, an editable transcript, and optional narration. Underneath, Storykept retains the provenance required to regenerate or export that story years later:

```text
original recording
→ original transcript
→ user corrections
→ analyzer state and tags
→ generated treatment(s)
→ fidelity report
→ family-facing story
```

That separation is the most important part of the engine. Models will change. Prompts will improve. A family memory should not become unrecoverable because the app once chose the wrong adjective—or the wrong provider.

The StoryEngine is useful precisely because it treats AI as replaceable processing around a durable human source.

---

*Storykept is private by default and built for family stories that should outlast the software used to record them. [storykept.app](https://storykept.app).*
