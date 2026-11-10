# BoxScoreBrief

**A sports-by-the-numbers Instagram account that runs itself, and an eval harness that keeps it honest.**

[@boxscorebrief](https://www.instagram.com/boxscorebrief/) posts NFL, NBA and MLB predictions every day.
My own models make the picks, Claude writes each caption from the numbers only, a fact checker blocks
invented stats, and the posts publish through Instagram's official API, all on a schedule, with no one at the keyboard.
Honesty is the brand: every miss gets posted, and the models are scored against the betting market in public.

## At a glance

| | |
|---|---|
| **Live since** | Oct 12, 2026 · 113 posts published automatically · 407 followers in 4 weeks |
| **What works on Instagram** | Reels reach ~10× more accounts than image posts (avg 94 vs. 9), so the schedule is Reels-first |
| **Caption quality** | 3.38 → **4.81** / 5 on held-out posts after eval-driven prompt fixes; "loser framing" 9/16 → **0/16** |
| **LLM judge you can trust** | Calibrated against my hand labels: Cohen's kappa 0.44 → **0.61**, factual agreement 29/30 |
| **Fact checker** | False alarms cut **67%** (15 → 5 of 30 labeled posts) |
| **Agent vs. prompt** | A self-revising agent matched the improved prompt's quality at **~40× the tokens**, so production uses the prompt |
| **Models vs. the market** | NFL 65.6% on 2,219 held-out games (market 66.2%); NBA 66.8% on 8,913 games. The market is still ahead, and the page says so |
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
   Every feature is recorded before the game is played, and a test mutates future data to prove nothing leaks.
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
| **Calibrate the LLM judge before trusting it** | Taking its grades at face value | Kappa against my hand labels started at only 0.44; reviewing every disagreement fixed errors on *both* sides (0.61) |
| **Split eval data by date, plus a daily check on brand-new posts** | A random split | Mirrors real use, and the daily job catches what a small test set can't (106 fresh posts scored so far) |
| **Claude over a local model** | Qwen 7B/3B on the laptop | 7B was accurate but took 77 s per caption vs. ~3 s; 3B was the most dangerous, with the right format but wrong facts |
| **Benchmark the models against the betting market** | Reporting accuracy vs. a coin flip | The market is the honest bar. It's still ahead, and the account says so |
| **Reels-first schedule** | Gut feel about formats | Insights showed ~10× the reach for Reels (avg 94 vs. 9 accounts) |
| **Original synthesized audio** | Licensed or trending sounds | Nothing to license, and nothing for Instagram to mute |

### The fact checker in code

The interesting part is not "find numbers that aren't in the facts." It's *which* numbers are legitimately new.
Each line below came from a real false alarm on a live draft:

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
    known = _facts_numbers(facts) | _known_percents(facts) | _list_lengths(facts) | _rounded(facts)
    bad = []
    for m in NUMBER.finditer(clean):
        n = m.group(0)
        if m.end() < len(clean) and clean[m.end()].isalpha():
            continue                                     # "76ers" is a team name, not a stat
        if n not in known:
            bad.append(n)
    return sorted(set(bad), key=float)
```

v1 → v2 cut false alarms on 30 hand-labeled posts from 15 to 5. v3 (shown above) clears all 23 posts that had been stuck,
and flags just 1 of 90 drafts and published posts, and that one is a real error.
**What it hasn't been tested on yet:** how many *invented* numbers it catches. That's the next experiment (see Limitations).

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
a "35.7%" I'd marked as invented was in the data (as 0.357), and it caught three real errors I'd missed that no number
checker could, because every number in them was correct (a false "biggest jump," a model adjustment described as a real score, an
invented reason for a pick). After fixing the rubric *and* my labels: **κ = 0.61, factual agreement 29/30.**

### What the measurements changed

| On 16 held-out test posts (date split) | Prompt v1 | Prompt v2 | Self-revising agent |
|---|---|---|---|
| Mean quality (1–5) | 3.38 | **4.81** | 4.81 |
| "Loser framing" | 9 / 16 | **0 / 16** | 1 / 16 |
| Judge-flagged factual errors | 2 / 16 | **0 / 16** | 0 |
| Input tokens per caption | ~2.4k | ~2.6k (+5%) | **~105k (~40×)** |

The worst habit, describing our own pick losing, came from **my own prompt**. Its example sentence
("a 60% favorite loses 4 times in 10") was being copied. Models treat examples as instructions.

Because some test posts inspired the fixes, a daily job also scores **every new post** with both prompts.
Across **106 fresh posts** (Oct 22 – Nov 10): quality **3.53 → 4.51**, loser framing **47 → 5**, non-factual **12 → 7**,
with no latency cost (median 4.6 s both).
