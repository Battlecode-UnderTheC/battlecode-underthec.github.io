---
layout: post
title: "Mass code generation"
description: "How we replaced one trained bot with hundreds of LLM-written variants, a local league to rank them, and what that league got wrong."
date: 2026-10-11 09:02:00 +0900
---

The idea came from NYPC, whose rules prohibit calling LLMs en masse in an automated way ([post 0]({% post_url 2026-10-11-0-vibe-coded %})). Our reading of the ban was short: if a competition bothers to ban it, it probably works, and there is published work behind the approach.

On 09-28 the [reinforcement learning plan]({% post_url 2026-10-11-1-rl %}) had just lost 79-1-0 to a scripted C++ bot written the same night. That bot became strategy-v1, our third server submission, and the starting point for everything below. We didn't improve it by hand because we didn't want to spend much of our own time: a loop could run without us.

## The bet

Instead of training one bot, we would have LLM agents write many bot variants, measure every one of them the same way, and breed from the winners. A human would write the shared layer (engine port, protocol parser, the agent interface) and the measuring tools. Agents would write strategies.

The plan is in [DESIGN.md](https://github.com/Battlecode-UnderTheC/syeyoung-tokens/blob/main/DESIGN.md) in syeyoung-tokens. Its first principle was "the league is the only judge": an agent never reports its own score. Its second was that nothing lives only in a terminal. Every match is a database row, every generation gets a committed report, and there are no command-line tuning knobs.

## The variant contract and the league

A variant was a directory, `variants/<name>/`, holding:

- the strategy as C++, written against a shared agent interface;
- `variant.toml` with its name, 0-2 parents, an operator (fresh, mutate, crossover or tweak), an archetype and the author;
- `NOTES.md`: the idea, what changed from the parents, and what the author expected, written before evaluation.

Its identity was an annotated git tag, `variant/<name>`, which never moved. `scripts/submit.sh` refused an existing name and refused any change outside the variant's own directory. The same source built two ways: a `.so` plugin loaded by our native simulator and the league, and a flat unswbc bot for the real judge.

The league service (`league/league.py`, 09-28) was standard-library Python with a SQLite database. Each pass it:

1. ingested new `variant/*` tags;
2. ran the gates: `check` on the public maps (no crash, no avoidable wall or self death), then at least 75% wins against a random anchor, both seats;
3. played matches six at a time, every pair on every public and rotating map from both seats;
4. refit Bradley-Terry ratings (draws as half a win, a weak prior), on an Elo scale with anchor-random fixed at 1000.

The maps came in suites. Public were the official maps. Rotating was 20 generated maps, replaced every 7 days. The holdout was 60 maps from the RL project's map pool plus 40 generated from a secret seed range. Agents never saw the holdout or any per-map result on it, and it never fed the rating. It was there to pick champions and to catch overfitting.

The whole thing ran on one M1 Pro Mac with 10 cores. The native engine played about a match per second per thread.

## Generations, then a continuous loop

Generation 1 started on 09-28 with strategy-v1 (Elo 1743), the two anchors, and six tasks. The orchestrator, a Claude Code session following [ORCHESTRATOR.md](https://github.com/Battlecode-UnderTheC/syeyoung-tokens/blob/main/ORCHESTRATOR.md), wrote one brief per task and launched one worker per brief in its own git worktree. Workers were Opus 5.5, plus Sonnet 5 for a pure parameter tweak. Three started together. The other three joined an hour later, when we lifted the cap of three, since workers mostly wrote code and needed little CPU.

The results came back that evening. counter-swarm-1 rated 1907, gather-split-trade-1 1892 and swarm-hunt-1 1873, all against strategy-v1's 1740. Each beat strategy-v1 45-0-17 or 44-0-18.

Midway through generation 1, we asked for a much shorter loop, because a generation took too long. From the evening of 09-28 tasks were capped at 20 minutes, six slots ran at once, and a new task started whenever a slot freed up, with no barrier between generations. The orchestrator still numbered tasks by generation, so "g14-9" is task 9 of generation 14. By g25 on 09-29 the gst line alone had reached gather-split-trade-131. The repository took 434 commits on 09-28 and 509 on 09-29, counting the workers' task branches (378 and 393 reached main).

How the agent loop itself worked, and how it changed, is [post 5]({% post_url 2026-10-11-5-llm-loop %}).

## The gst line

gather-split-trade-1 ("gst1") was strategy-v1 plus a look-ahead that simulated bodies and pearl-target sharing between allies. In generation 2, gst4 added a champion that all dragons agreed on over sonar and fed in the endgame. That went 29-11 against its parent and rated about 70 Elo higher. Most of what came later in the line was small additions on that base: cap-aware trap safety (gst27), a portal fix, partial sightings and obituaries for dead champions, a centre pull on small maps, and pricing dead ends by expected loss so short dragons could farm corridors and split out (gst96).

Other lines ran beside it: counter-swarm, swarm-hunt, body-wall, late-surge, giant, territory, portal-farmer. They traded ideas freely. A gst fix was often ported into three other lines within the hour. gst stayed on top. What the lines actually did is in [post 6]({% post_url 2026-10-11-6-bots %}).

We submitted to the competition server from the league's top. We approved gst27 as v4 on 09-28 around 20:40. From 09-29 we had standing authority to submit the best bot every four hours:

| server version | variant | submitted | evidence at the time |
|---|---|---|---|
| v4 | gather-split-trade-27 | 09-28 ~20:40 | judge sandbox 2-1 vs strategy-v1 (v3) |
| v5 | gather-split-trade-51 | 09-29 02:54 | league #1, judge sandbox 10-1 vs v4 |
| v6 | gather-split-trade-90 | 09-29 06:54 | league #1, sandbox 12-10 vs v5 |
| v7 | gather-split-trade-96 | 09-29 17:57 | league #1, sandbox 5-3 vs v6 |
| v8 | gather-split-trade-131 | 09-30 01:01 | sim 287-233 vs v7, judge sandbox 24-20 |

The sandbox evidence behind each was a handful of games. The league rating carried the decision, and the last section of this post is about why that was a problem.

## The first server battles

The first server battle went 5-5 against a trading swarm: no sonar, always `SPLIT 2`, 3-5-long dragons lunging 2-4 steps at our long ones. Our dragons spent 8.7% of their moves in idle loops, 75% of those orbiting portals. The second we lost 2-8 to a stronger version of the same shape that crossed portals 1,098 times and farmed portal-only food (709 pearls to our 8 on Trauma).

<figure markdown="1">
<img src="{{ '/assets/img/film-first-battle.webp' | relative_url }}" alt="Our champion dying head-on in the first server battle" loading="lazy">
<figcaption markdown="span">The first server battle, on Autarky, rounds 390 to 395 (ours in cyan). Our 21-long champion meets a 5-long enemy head-on, both die, and its length is left on the board as pearls.</figcaption>
</figure>

That evening produced a portal fix (gst33 and its ports into the other lines) and a new kind of variant: clones of real opponents, starting with trading-swarm-1. The clones mattered because A/B tests against our own parents couldn't see these weaknesses at all.

## What the league got wrong

**Whose side we were on.** For about a day we read server replays with the sides swapped. The replay's `botA`/`botB` fields hold *our* submission id, and the opponent's is blank. The "opponent" diving 1-wide columns on Devil was our own bot, in a win. On 09-29 we corrected the record: the real series against the current opponents was 7-18, not 13-7, and trading-swarm-6 was a clone of our own behaviour. g19's premise was wrong and three of its tasks were dropped. The fix in KNOWLEDGE.md was a rule: identify our team by our own sonar words, never by the name.

**Sim versus judge.** By g25 (09-29 evening) the numbers stopped agreeing. Every variant since v7 that beat it 55-60% in our simulator or the league scored 19-24 out of 44 against it in the judge's sandbox: gst100, 101, 115, 127, 128 and 131. Even at the same seed, single games differed. Until it was fixed, verdicts moved to multi-seed sandbox duels (`scripts/sbx-duel.sh`), and a tooling task was set to make the simulator reproduce the judge.

The cause was found on 10-01. Apple clang on arm64 compiles with `-ffp-contract=on` by default, so our native plugins fused `a*b+c` into one FMA instruction. The judge runs bots as WebAssembly, which never fuses. Floating-point scores differed in the last bit, ties broke the other way, and games diverged. Before the fix, gst-queen-1 against gst131 matched the judge in 18 of 30 games in one set and 13 of 20 in another. After compiling everything with

```
-ffp-contract=off
```

they matched 30 of 30 and 20 of 20. An earlier theory, shared mutable state in the bots, had been wrong. Every sim number for the score-based lines (gst, search, rollouts) from before the fix carried this noise. That includes the evidence behind v5 to v8, although each of those also had sandbox games.

The league was the only judge by design, and for those days it had been judging a slightly different game. On 10-02 we stopped running it and moved every verdict to [the team arena]({% post_url 2026-10-11-3-arena %}), which plays only the official judge. By then the local league database held 170,039 matches between 206 variants.
