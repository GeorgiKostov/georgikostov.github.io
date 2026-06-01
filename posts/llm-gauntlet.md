---
layout: default
title: "The 25-model gauntlet: picking the LLM that interviews your grandparents"
date: 2026-05-31
description: "What a voice-first memoir app learned benchmarking ~25 LLMs across 7 languages for speed, cost, and quality."
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

## What I shipped

I kept **Groq Llama 3.3 70B** in production. It's the fastest by a wide margin, its quality gaps close with prompting (and I closed them), and it runs on the same provider as my speech-to-text, so there's one less network hop. Claude Haiku 4.5 is the documented fallback when I want maximum extraction quality and can spend the latency. Gemini 2.5 Flash is the balanced option if I ever switch providers.

The bigger keeper was the harness. It lists every provider's current models, scores latency, cost and per-language quality, and runs a dirty-content tier that already caught one shipped bug and saved me from one bad swap. Next time a model drops, picking it is one command instead of three more days.

If you're putting an LLM on a hot path, build the mean little benchmark first. Test it dirty, turn the thinking off, and measure speed where the user actually feels it.

---

*Storykept records family stories by voice. Private by default, built to outlast its users, tested in Bulgarian because that's the language my dad tells his stories in. [storykept.app](https://storykept.app).*
