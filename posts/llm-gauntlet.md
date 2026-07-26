---
layout: default
title: "The 25-model gauntlet: picking the LLM that interviews your grandparents"
header_title: "The LLM gauntlet"
date: 2026-05-31
updated: 2026-07-26
description: "What a voice-first memoir app learned from two LLM benchmarks, seven languages, and a month of production latency."
author: George Kostov
permalink: /posts/llm-gauntlet/
---

# The 25-model gauntlet: picking the LLM that interviews your grandparents

I'm building [Storykept](https://storykept.app), a voice-first app that records family stories. You talk, an AI listens, works out what's missing from the memory, and asks one good follow-up: "what year was that?", "who else was there?" Then it stitches the answers into a clean chapter.

The part doing the listening is what we call the analyzer. After every spoken turn it grades how complete the story is, pulls out the people, places and dates, and writes the next question, all as strict JSON. It fires on *every* turn, while a 78-year-old sits there waiting for a reply. So the model behind it has to clear three bars at once:

- **Fast.** Latency is felt live, not in a benchmark.
- **Multilingual.** My dad, our first real user, tells his stories in Bulgarian.
- **Reliable on messy speech.** Real talk is full of "no wait, it was sixty-seven, not sixty-five."

Picking it took a few days and about 25 models across five providers. Here's what shook out.

<div class="article-lead">
  <p><strong>Updated July 2026:</strong> the May benchmark below is still the experiment we ran, but it is no longer the end of the story. Production traffic exposed a long-context latency problem, we rebuilt the benchmark around the exact shipped prompt, and Claude Haiku 4.5 replaced the original winner. I have kept the first result intact and added what changed after we had real users waiting.</p>
</div>

## A bench, not a vibe check

A clean transcript ("I was born in 1952 in Vienna") tells you nothing. Every model handles that. So I wrote deliberately *dirty* transcripts in seven languages, each carrying:

- a spoken self-correction ("nineteen sixty-five, no wait, sixty-seven")
- one person with two names ("my brother Tomas, uh, Tomash")
- age wobble, disfluencies, trailing "ums"

A model passes a cell only if it keeps the corrected value (1967, never 1965), merges the name and nickname into one person, and answers in the storyteller's language. Bulgarian is a hard gate. If it can't hold my dad's story in Bulgarian, it doesn't ship.

The harness runs the same prompt against any model and logs latency, token cost, and the raw JSON, so I can read the outputs side by side instead of trusting a leaderboard.

## Speed, by language

This is the table that decided it. Median latency, milliseconds, seven languages:

| Model | EN | BG | DE | ES | FR | RU | ZH |
|---|---|---|---|---|---|---|---|
| **Groq Llama 3.3 70B** | 565 | 645 | 559 | 459 | 552 | 638 | 545 |
| Gemini 2.5 Flash | 1898 | 1527 | 1329 | 1455 | 1497 | 1434 | 1384 |
| gpt-4.1-nano | 1311 | 1319 | 1312 | 1101 | 970 | 1192 | 963 |
| gpt-4o-mini | 1866 | 2033 | 2019 | 1816 | 1849 | 1831 | 2415 |
| Claude Haiku 4.5 | 3113 | 3688 | 3333 | 2568 | 3015 | 2957 | 2705 |

Groq sits in its own column. It's 2x faster than the next thing and 5x faster than Claude, and it doesn't wobble by language. When the call fires on every turn, the gap between 570ms and 1.5s is the difference between "the app is listening" and "the app is buffering." (Gemini 3.1 Flash-Lite isn't in the table because its latency swung from 1.2s to 7s across runs. Too jumpy to trust live.)

## Price

All of them are cheap. The per-call cost of the analyzer rounds to nothing next to the speech-to-text and the final polish:

| Model | Cost / call | Cost / 1,000 calls | Avg latency |
|---|---|---|---|
| gpt-5-nano | $0.00013 | $0.13 | ~2,100ms |
| gpt-4.1-nano | $0.00014 | $0.14 | ~1,170ms |
| Gemini 3.1 Flash-Lite | $0.00017 | $0.17 | spiky |
| gpt-4o-mini | $0.00021 | $0.21 | ~1,980ms |
| Groq Llama 3.3 70B | $0.00060 | $0.60 | ~570ms |
| Gemini 2.5 Flash | $0.00078 | $0.78 | ~1,500ms |
| Claude Haiku 4.5 | $0.00340 | $3.40 | ~3,050ms |

The cheapest model is ~25x cheaper than the priciest, and across a whole recording that's a difference of fractions of a cent. Price didn't make the decision. I list it so you can see it stop mattering.

## Quality, by language

Pass marks on the dirty transcripts. ✅ clean (dropped the mistake, merged the names, stayed in language), 🟡 correct but rough, ⚠️ a real miss (kept the wrong value, or split one person into two):

| Model | EN | BG | DE | ES | FR | RU | ZH |
|---|---|---|---|---|---|---|---|
| Groq Llama 3.3 70B † | ⚠️ | ✅ | ⚠️ | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| Gemini 2.5 Flash | 🟡 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| gpt-4.1-nano | ⚠️ | ⚠️ | ✅ | ⚠️ | ✅ | ✅ | ✅ |
| gpt-4o-mini | ⚠️ | ✅ | 🟡 | ⚠️ | ⚠️ | ✅ | ⚠️ |
| Claude Haiku 4.5 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

† That row of red is on the *bare* test prompt, with no dedup instruction. More on that below, because it's the whole twist.

Two things jump out. Claude Haiku is the best extractor of anything I tested, in every language. And the cheap-and-fast tiers (gpt-4.1-nano, gpt-5-nano) leak: they kept the wrong year on three of seven languages. Cheap and unreliable loses to nothing.

## Four things the bench taught me

**1. Reasoning models are a trap for this job.** Our first analyzer was `gpt-5-nano`, and it ran ~16 seconds a turn because it's a reasoning model burning thousands of thinking tokens on what is basically classification. Gemini bit me the same way: `gemini-3.5-flash` looked terrible (5-8s, broken JSON) until I set `reasoning_effort: 'none'` and it dropped to ~1.1s and clean. If you put a reasoning-capable model on a fast structured task, turn the thinking off before you judge it. OpenAI's gpt-5 line has the same switch, with a trap: `gpt-5.1` dropped the `'minimal'` value the others use and only takes `'none'`, which 400'd every call until I caught it.

**2. Clean benchmarks lie.** On an earlier, tidier test set, Llama 4 Scout looked like the winner: fastest, cheapest, deduped names on its own. Then I ran it through the dirty transcripts, with stacked corrections and two different people both named George. It split "Harold/Harry" into two people and kept both the wrong and right value on corrections. It scored 2 out of 5. The tidy benchmark had hidden the exact failure that matters for real speech.

**3. The bench found a bug in my own production prompt.** Testing the challengers, I noticed my *shipped* analyzer did the same thing: given "we lived in Перник, no, in Кремиковци," it saved both cities. My prompt literally said "never the mistaken one," but the models honored that for a lone name or year and dropped it on places and stacked corrections. Those entities become the tags on someone's family story. I rewrote the instruction to be specific about places, streets and counts, added a final re-scan, and the leak closed. A mean benchmark is a bug-finder for your own system, not just a model picker. That red row above is Llama without the fix. With the production prompt, it deduplicates and corrects cleanly.

**4. Cap your agents.** I run a lot of this through coding agents, and I let one off the leash with "if it errors, retry and debug" and no time limit. A thinking-model JSON failure sent it into a 35-minute loop writing probe scripts before I killed it. Now every run has a hard timeout and a per-call abort, and no agent gets "debug until it works." An agent without a time budget doesn't stop on its own.

## What I shipped in May

I kept **Groq Llama 3.3 70B** in production. It's the fastest by a wide margin, its quality gaps close with prompting (and I closed them), and it runs on the same provider as my speech-to-text, so there's one less network hop. Claude Haiku 4.5 is the documented fallback when I want maximum extraction quality and can spend the latency. Gemini 2.5 Flash is the balanced option if I ever switch providers.

The bigger keeper was the harness. It lists every provider's current models, scores latency, cost and per-language quality, and runs a dirty-content tier that already caught one shipped bug and saved me from one bad swap. Next time a model drops, picking it is one command instead of three more days.

If you're putting an LLM on a hot path, build the mean little benchmark first. Test it dirty, turn the thinking off, and measure speed where the user actually feels it.

## July update: production changed the question

The original gauntlet sent short, deliberately dirty fixtures through a common task. That was the right test for multilingual extraction. It did not reproduce the thing that became expensive in production: on every turn, the analyzer was receiving the entire interview again.

A 30-day snapshot from `storykept_ai_calls` made the tail visible:

| Production analyzer metric | Before the redesign |
|---|---:|
| Calls in the snapshot | 113 |
| Median latency | 4.0s |
| p95 latency | 10.3s |
| Slowest call | 26.7s |
| Average input | 5,185 tokens |
| Average output | 172 tokens |
| Total model cost | $0.10 |

Cost was still noise. Waiting was not. The median was survivable, but a 10-second p95 breaks the rhythm of an interview.

So the second benchmark stopped asking, “Which model extracts this fixture fastest?” It imported the byte-identical production prompt and graded the complete job: question craft, story-type classification, entity retention, median latency, worst-case latency, and cost. An Opus judge scored the follow-up questions without knowing which model wrote them.

The definitive run used four passes across eight fixtures and all nine story types. **Claude Haiku 4.5 won both main axes:** a 2,201ms median and 4.14/5 question quality. It retained all required entities in 8/8 fixtures, classified 7/8 correctly, and kept its worst case to 7.3 seconds. A model with a slightly tempting median was rejected because its tail reached 16 seconds. On a conversational hot path, the worst wait matters almost as much as the typical one.

That result looks inconsistent with the May table, where Haiku averaged about three seconds and Groq dominated. It is not. The prompts, fixtures, judging, and production constraints changed. The first benchmark found the fastest acceptable extractor. The second found the best interviewer running the code we actually shipped.

## The model swap was only half the fix

Changing providers did not solve the structural problem of resending an ever-growing interview. We added a small persisted analyzer state:

- a faithful running summary of the story so far
- six completeness grades: setting, people, setup, action, outcome, and significance
- the turn count
- the people, places, dates, and themes already established

After a story passes eight turns, the next call receives that state plus the last three exchanges instead of the complete transcript. The full entity set, prior questions, storyteller coverage, and family roster still travel separately, so the model does not forget an early name just because the raw line has left the window.

We ran the windowed and full-history paths against long English and Bulgarian fixtures of 16–26 turns. They produced identical six-axis completeness, identical completion decisions, byte-identical next questions, full entity retention, and a tied blind quality judgment. Tokens fell by about 10% at those fixture lengths, with the saving growing as the story gets longer. Only then did we turn the windowed path on by default, with a kill switch left in place.

The static instructions are also prompt-cached, and the live chain is now:

```text
Claude Haiku 4.5
        ↓ provider failure
OpenAI gpt-4o-mini
```

Groq was removed from the production chain in July as a provider decision, not because the May timing numbers were false. The cross-provider fallback is deliberate: an Anthropic outage should not take the interviewer down with it.

## The lesson after two benchmarks

An LLM benchmark is not a permanent ranking. It is an executable description of what you currently care about.

In May, we cared about multilingual correction handling and sub-second response. In July, real sessions taught us to care about question craft, classification, retained context, and tail latency under a growing prompt. The harness had to evolve with the product.

That is the result I trust now: not “Haiku is the best model,” but “Haiku is the best current fit for this exact production contract, and the contract is measured well enough that we can change our mind again.”

---

*Storykept records family stories by voice. Private by default, built to outlast its users, tested in Bulgarian because that's the language my dad tells his stories in. [storykept.app](https://storykept.app).*
