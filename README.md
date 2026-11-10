# BoxScoreBrief

**A sports-by-the-numbers Instagram account that posts itself, and an eval harness that keeps it honest.**

[@boxscorebrief](https://www.instagram.com/boxscorebrief/) posts NFL, NBA and MLB predictions almost every day.
My own models make the picks, Claude writes each caption from the numbers only, a fact checker blocks
invented stats, and the posts publish through Instagram's official API on a schedule. Only posts the checker flags wait for my review.
Honesty is the brand: every miss gets posted, and the models are scored against the betting market in public.

<p align="center">
  <img src="assets/odds-race-demo.gif" width="270" alt="A BoxScoreBrief Reel: Milwaukee's World Series odds climbing from 26% to 100%">
  <br><sub>The account's top Reel (264 accounts reached): title odds drawn over the season, rendered in Python with original sound.</sub>
</p>

## At a glance

| | |
|---|---|
| **Live since** | Oct 12, 2026 · 113 posts published by Nov 9 (106 with no human step) · 407 followers in 4 weeks |
| **What works on Instagram** | Reels reach ~10× more accounts than image posts (avg 94 vs. 9), so the schedule is Reels-first |
| **Caption quality** | 3.38 → **4.81** / 5 on held-out posts after eval-driven prompt fixes; "loser framing" 9/16 → **0/16** |
| **LLM judge, validated** | Checked against my hand labels: Cohen's kappa 0.44 → **0.61** (30 posts), factual agreement 29/30 |
| **Fact checker** | False alarms cut **67%** (15 → 5 of 30 labeled posts); catches ~**70%** of 4,596 planted wrong numbers |
| **Agent vs. prompt** | A self-revising agent matched the improved prompt's quality at an estimated **~40× the tokens**, so production uses the prompt |
| **Models vs. the market** | Held-out games: NFL 65.6% vs. market 66.2% (2,219 games); NBA 67.6% vs. 68.8% (3,961 games). The market is still ahead, and the page says so |
| **Engineering** | Python · FastAPI · Docker Compose · APScheduler · Claude API · 300+ tests across 4 repos |

## How it works

```mermaid
flowchart LR
    subgraph Models["Prediction models (my code)"]
        NFL["NFL<br/>ridge · injuries"]
        NBA["NBA<br/>ridge · player availability"]
        MLB["MLB playoffs<br/>logistic · exact series odds"]
    end
    Data[(Public sports data<br/>+ betting lines)] --> Models
    Models --> Story["Story builder<br/>facts + card / Reel spec"]
    Story --> Writer["Claude writer<br/>structured output,<br/>numbers only"]
    Writer --> Check{"Fact checker<br/>every number traced<br/>to the facts"}
    Check -- clean --> Render["Render<br/>Win95 cards · Reels<br/>(Pillow + ffmpeg, original audio)"]
    Check -- flagged --> Human["Human review<br/>approve / edit / reject"]
    Human --> Render
    Render --> Sched["Scheduler"] --> IG(["Instagram<br/>official API"])
    IG --> Insights[(Insights:<br/>reach · views · followers)]
    Insights -. "Reels-first schedule" .-> Sched
    Posts[(Every post:<br/>facts + caption)] --> Evals["Eval harness<br/>rule graders + calibrated LLM judge"]
    Writer --> Posts
    Evals -. "prompt fixes" .-> Writer
```

1. **Predict.** Three models I built and backtested walk-forward: NFL and NBA ridge models with injury and
   player-availability features, and an MLB playoff model that computes exact series and bracket odds.
   Every feature is computed only from data available before the game, and a test mutates future data to prove nothing leaks.
2. **Tell the story.** A source module turns each prediction into a `Story`: the exact facts a caption may use,
   plus the card or Reel to draw.
3. **Write.** Claude writes the headline, caption and hashtags from those facts only, returned as structured output.
4. **Check.** A fact checker traces every number in the caption back to the facts, allowing derived numbers
   ("a 66% favorite loses about 1 in 3") but not invented ones. Clean posts publish automatically; flagged posts
   wait for me on a review dashboard.
5. **Render and publish.** Win95-style images and 1080×1920 Reels are drawn in Python with original synthesized
   sound, then scheduled and published through Instagram's official API.
6. **Measure.** Insights flow back every 6 hours (Reels proved to have ~10× the reach, so the schedule changed),
   and the eval harness grades captions with rule checks and an LLM judge calibrated against my own labels.

## Key decisions

| Decision | Instead of | Why |
|---|---|---|
| **Captions may only use numbers from a facts object** | Letting the model reason over raw data | It defines exactly what's allowed to be said, which makes every claim checkable |
| **A cheap rule-based fact checker gates publishing; flagged posts go to a human** | An LLM judge in the hot path, or blocking anything flagged | Fast, free and deterministic. When the old checker blocked flags automatically, 23 accurate posts got stuck, so flags became a review queue |
| **A one-shot prompt in production, not an agent** | A self-revising agent (draft → check → revise) | The agent tied the improved prompt on quality (4.81) at ~40× the tokens. Rule: start cheap, escalate only when a cheap check fails |
| **Validate the LLM judge against my labels before trusting it** | Taking its grades at face value | Kappa against my hand labels started at only 0.44; reviewing every disagreement fixed errors on *both* sides (0.61) |
| **Split eval data by date, plus a daily check on brand-new posts** | A random split | Mirrors real use, and the daily job catches what a small test set can't (106 fresh posts scored so far) |
| **Claude over a local model** | Qwen 7B/3B on the laptop | 7B was accurate but took 77 s per caption vs. ~3 s for Claude in the same test; 3B was the most dangerous, with the right format but wrong facts |
| **Benchmark the models against the betting market** | Reporting accuracy vs. a coin flip | The market is the honest bar. It's still ahead, and the account says so |
| **Reels-first schedule** | Gut feel about formats | Insights showed ~10× the reach for Reels (avg 94 vs. 9 accounts) |
| **Original synthesized audio** | Licensed or trending sounds | Nothing to license, and nothing for Instagram to mute |

### The fact checker in code

The interesting part is not "find numbers that aren't in the facts." It's *which* numbers are legitimately new.
Each line below came from a real false alarm on a real caption:

```python
def unsupported_numbers(text: str, facts: dict) -> list[str]:
    """Numbers in the text that don't appear anywhere in the facts."""
    clean = re.sub(r"#\w+", "", text)                    # ignore hashtags (#49ers)
    clean = re.sub(r"(?<=\d),(?=\d{3}\b)", "", clean)    # "11,437" is one number
    clean = LABEL.sub("", clean)                         # "Slide 1" is a label, not a stat
    rates = _known_rates(facts)
    for match in ODDS.finditer(clean):                   # "1 in 3" derived from a 66% favorite
        if _is_derived(match.group(1), match.group(2), rates):
            clean = clean.replace(match.group(0), "")
    for match in TIMES.finditer(clean):                  # "20 times the usual rate" (18% vs 0.9%)
        if _is_ratio(match.group(1), facts):
            clean = clean.replace(match.group(0), "")
    known = (_facts_numbers(facts) | _known_percents(facts) | _list_lengths(facts) | _rounded(facts)
             | _series_differences(facts))           # a team's own rise: 26% -> 100% makes "74" valid
    bad = []
    for m in NUMBER.finditer(clean):
        n = m.group(0)
        if m.end() < len(clean) and clean[m.end()].isalpha():
            continue                                     # "76ers" is a team name, not a stat
        if n not in known:
            bad.append(n)
    return sorted(set(bad), key=float)
```

v1 → v2 cut false alarms on 30 hand-labeled posts from 15 to 5. v3 clears all 23 posts that had been stuck, and flags
just 1 of 90 drafts and published posts, and that one is a real error. v4 (shown above) allows a team's *own* rise or fall
(26% → 100% is "rose 74 points"), which the new prompt asks for, and nothing broader: on title-odds captions it took false
flags from 8 of 15 to 0, at a cost of under one point of overall recall (4 points on title-odds captions).

**How many invented numbers does it catch?** I planted 4,596 single-number errors (±1, +7, ×1.5) into 114 real captions:
it catches about **70%**. The misses are mostly numbers that happen to exist elsewhere in the facts, which is why the
LLM judge exists (see Limitations).

## How I know it works: the eval harness

[boxscorebrief-evals](https://github.com/jddormer19/boxscorebrief-evals) grades every caption two ways:

- **Rule graders** (plain code): every number supported, under 900 characters, ends with a question, a headline, emoji and betting-language rules.
- **An LLM judge** (Claude, structured output): factual or not, "loser framing" or not, and quality 1–5, with hard caps:
  any invented stat caps quality at 1.

An LLM judge is only useful if it agrees with a human, so I hand-labeled 30 posts and measured agreement before trusting it:

```python
class Verdict(BaseModel):          # the judge must answer in this shape
    factual: bool
    loser_framing: bool
    quality: int                   # 1-5; an invented stat caps it at 1
    reason: str

def kappa(human, model):
    """Cohen's kappa: agreement between two lists of labels, corrected for chance."""
    n = len(human)
    observed = sum(h == m for h, m in zip(human, model)) / n
    categories = set(human) | set(model)
    expected = sum((human.count(k) / n) * (model.count(k) / n) for k in categories)
    if expected == 1:
        return 1.0
    return (observed - expected) / (1 - expected)
```

The first run agreed with me at only **κ = 0.44**. Going through all 9 disagreements, the judge was right more than once:
a "35.7%" I'd marked as invented was in the data (as 0.357), and it caught three real errors I'd missed (two misread claims and
one invented reason) that no number checker could, because every number in them was correct: a false "biggest jump," a model
adjustment described as a real score, and an invented reason for a pick. After fixing the rubric *and* my labels: **κ = 0.61, factual agreement 29/30.**

### What the measurements changed

| On 16 held-out test posts (date split) | Prompt v1 | Prompt v2 | Self-revising agent |
|---|---|---|---|
| Mean quality (1–5) | 3.38 | **4.81** | 4.81 |
| "Loser framing" | 9 / 16 | **0 / 16** | 1 / 16 |
| Judge-flagged factual errors | 2 / 16 | **0 / 16** | 0 |
| Input tokens per caption | ~2.4k | ~2.6k (+6%) | **~105k (~40×, estimated)** |

The worst habit, describing our own pick losing, came from **my own prompt**. Its example sentence
("a 60% favorite loses 4 times in 10") was being copied. Models treat examples as instructions.

Because some test posts inspired the fixes, a daily job also scores **every new post** with both prompts.
Across **106 fresh posts** (Oct 22 – Nov 10): quality **3.53 → 4.51**, loser framing **47 → 5**, non-factual **12 → 7**,
with no latency cost (median 4.6 s both). One thing got worse: v2 passed every rule grader less often (68 vs. 76 of 106),
mostly from title-odds captions that now compute rises and falls the checker didn't yet recognize, and more captions
missing the closing question.

**Shipping it, two weeks late.** v2 won the eval, but it stayed in the eval repo: production kept running v1 until
Nov 10, and 31% of captions published in the two weeks before used loser framing (19 of 61). v2 is now live, a test fails if
production's prompt ever drifts from the validated one, and the before/after on real published posts is being measured.

## The models, scored against the market

| | Held-out backtest | Model | Betting market | Live this season |
|---|---|---|---|---|
| **NFL** | 2018–2025 · 2,219 games | 65.6% · log loss 0.627 | 66.2% · 0.609 | 38 vs. market 41 of 56 matched games |
| **NBA** | 2020–2026 · 3,961 games with a market line | 67.6% · log loss 0.590 | 68.8% · 0.583 | 78 vs. market 84 of 118 |
| **MLB** | 2021–2025 · 12,356 games | 57.3% · log loss 0.677 | 57.9% · 0.674 (on 11,437 games with a line) | Postseason: 19–17 on games, 8 of 10 series |

Every backtest is walk-forward: the model only ever sees games before the one it predicts, and a test mutates future
data to prove nothing leaks. All three beat the simplest baseline (always picking the home team: 54.4% NFL, 55.3% NBA, 53.0% MLB),
**but none beats the betting market.** When my model and the market disagreed this season (NFL and NBA), the market was right 13 of 17 times.
That's why the account explains *why* a team is favored, and why it never gives betting advice.

## What went wrong (and what I changed)

- **The media tunnel dropped overnight on day one.** → A watchdog now checks it every 60 seconds and restarts it, and every post
  checks its media link is reachable before publishing.
- **The fact checker blocked accurate posts.** Its false alarms left 23 correct drafts stuck. → Rebuilt twice with
  measured false-alarm rates, and flags became a human review queue instead of a block.
- **My own prompt taught the model a bad habit** ("loser framing"). → Fixed in the eval repo and measured (9/16 → 0/16),
  but I didn't notice for two weeks that production was still on the old prompt. → Shipped Nov 10, with a parity test so
  production and the validated prompt can't drift apart again.
- **A title-odds Reel crashed when a team reached 100%.** → Axis capped, with a regression test.
- **A home/away mix-up briefly showed the World Series at 3–2.** It was caught before posting, and now there's a test.
- **The agent's workspace leaked into its results twice** (old captions, leftover drafts). Lesson: an agent's
  files are part of its input, so evals need a clean room.
- **A change shipped with a failing test.** → Every merge now gets a short written review.

## Limitations

- **Small samples.** 30 hand-labeled posts, 16 test posts, and post-format decisions based on as few as 1–6 posts per image type.
  The daily fresh-post job exists to grow the evidence.
- **The fact checker catches about 70% of planted wrong numbers.** Swapped teams and invented reasons get past it by design,
  which is what the LLM judge is for, and the judge isn't in the publishing path yet.
- **Kappa is unweighted** on a 1–5 scale, so it treats a 4-vs-5 disagreement like 1-vs-5.
- **Reels lose viewers in 2–4 seconds** of ~10, so reach is good but watch time isn't yet.
- **It runs on one laptop.** If the laptop sleeps, game-day posts expire.
- **CI covers only the eval repo,** and production doesn't yet log token cost per post.

## What's next

1. **Planted-error test for the LLM judge too**, then put the judge in the publishing path for posts the checker can't verify.
2. Weighted kappa alongside unweighted; CI for every repo; cost per post logged in production.
3. Move hosting off the laptop.
4. **The next project:** a research agent for Kalshi sports markets that uses these same models, built evals-first and
   benchmarked against the market.

## What this project demonstrates

- **LLM application engineering:** generation grounded in a facts object, structured outputs, a rule-based guardrail on
  every caption, a human review queue for anything flagged, and a measured prompt-vs-agent cost decision.
- **Evaluation:** rule graders plus an LLM judge validated against hand labels (Cohen's kappa), a date-based held-out split,
  a daily eval on brand-new posts, and planted-error tests: 4,596 wrong numbers planted into 114 real captions to
  measure how many the checker catches.
- **Applied ML and forecasting:** ridge and logistic models with injury and player-availability features, walk-forward
  backtests, leakage tests that mutate future data, exact series and bracket probabilities, and honest benchmarking against the betting market.
- **Production engineering:** Docker Compose services, a scheduler, Instagram's official API, a tunnel watchdog,
  pre-publish media checks, a parity test that keeps production on the validated prompt, rollback images, and a written review for every change.
- **Ownership:** I came up with it, built it, and run it: models, pipeline, evals, design and the account itself.

## Links

- **Live account:** [@boxscorebrief](https://www.instagram.com/boxscorebrief/)
- **Eval harness (public):** [boxscorebrief-evals](https://github.com/jddormer19/boxscorebrief-evals) ·
  write-up: [What my AI judge taught me about my own mistakes](https://github.com/jddormer19/boxscorebrief-evals/blob/master/WRITEUP.md)
- **Posting app and models:** private repos; walkthroughs available on request.

## Data sources and credits

- **NFL:** schedules, play-by-play, injuries and rosters from [nflverse](https://github.com/nflverse/nflverse-data),
  licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- **NBA:** public box scores, schedules and injury reports, used for this non-commercial project; to be replaced by a
  licensed feed before any commercial use.
- **MLB:** MLB data © MLB Advanced Media, used under its terms for non-commercial purposes.
- **Betting lines:** used only as a benchmark; aggregate accuracy is published, never raw odds.
- Captions are written by Claude (Anthropic) from my own models' outputs. All graphics and audio are original.

*BoxScoreBrief is an independent fan project, not affiliated with or endorsed by the NFL, NBA, MLB or any team.
Predictions are for entertainment, not betting advice.*
