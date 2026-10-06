# Metrics

How each number is computed, and what to make of it.

Definitions use these terms:

- **answer** - one response from one engine to one prompt in one repeat
- **unbranded answer** - an answer to a prompt whose category is anything except `brand_research`
- **branded answer** - an answer to a `brand_research` prompt

---

## Visibility

```
visibility = answers that mention the brand / unbranded answers
```

Computed per engine. The headline number in the report is the mean across engines.

This is the metric that matters. It answers: when a buyer asks a question that does not name the brand, does the engine bring the brand up?

Branded answers are excluded on purpose. A question containing the brand name will produce an answer containing the brand name on every engine, every time. Including those would lift every score by the same amount and hide the differences between engines.

---

## Brand knowledge

```
brand knowledge = branded answers that mention the brand / branded answers
```

What the engine knows when asked directly. Expect close to 100%. A value well below that means the engine has little or no information about the brand, which is a different and more basic problem than low visibility.

Read the two together:

| Visibility | Brand knowledge | Meaning |
|---|---|---|
| low | high | The engine knows the brand and does not recommend it. Work on presence in cited sources. |
| low | low | The engine barely knows the brand. Work on basic entity presence first. |
| high | high | Healthy. Watch for competitors catching up. |

---

## Citation rate

```
citation rate = answers citing the brand's domain / all answers
```

Whether the engine links to the brand's own site. Grounded engines (Perplexity, and the others with web search on) return sources; the rate is the share of answers where one of them is the brand's domain.

A brand can be mentioned without being cited, and cited without being recommended. The three are separate.

---

## Share of voice

```
share of voice = brand mentions / (brand mentions + competitor mentions)
```

Across all answers, per engine. Kept for history and for the alert on lost citations. Visibility is the number to read; share of voice mixes branded and unbranded answers and is less clean.

---

## Competitor board

For unbranded answers only:

```
rate = answers mentioning the name / unbranded answers
```

One row per listed competitor and one for the brand, sorted by rate. It shows who the engines recommend in place of you, and by how much.

Adding a competitor changes the picture. A weak name inflates the apparent standing of the brand; leaving out a strong one deflates it. Three to six is the workable range.

---

## Gap sources

Domains cited in unbranded answers where **at least one competitor is mentioned and the brand is not**.

These are the places where engines learn what to recommend, and the brand is absent from them. This is the actionable output of the whole measurement.

Each source is counted once per answer, so a page cited three times inside one response still counts once.

### Accumulated over a window

A single run's list is unstable, because engines cite a shifting set of pages. So gap sources are also aggregated over the last 28 days:

| Field | Meaning |
|---|---|
| `answers_total` | answers citing the domain, summed over the window |
| `runs_seen` | number of runs the domain appeared in |
| `persistence` | `runs_seen` / runs in the window |
| `gap_runs` | runs where it was a gap source |

A domain is **stable** when it has at least one gap run and `persistence` is 0.5 or higher. The recommendation step uses stable domains. When no domain qualifies yet, which is normal in the first weeks, it falls back to ranking by volume and says so.

### Competitor-owned domains

If a domain belongs to a listed competitor, it is marked as such before it reaches the recommendation step. Engines citing `competitor.com` is evidence that the brand needs its own comparison pages. It is not a place to pitch.

---

## Google vs AI

For each unbranded prompt, one search against classic Google results.

```
google presence = prompts where the brand is in the top 10 / prompts checked
```

Also reported: the average position among prompts where the brand appears at all.

This block matters most when the two disagree. A brand that ranks in Google and is absent from AI answers has a specific problem to solve. When both are zero, the report says so plainly and the block adds less.

---

## Week-over-week change

Per engine, the change in share of voice against the most recent earlier run, shown in percentage points.

The comparison is made only against runs with the same `prompt_set_version`. When the question set changes, the trend restarts.

### Alerts

An alert fires when either holds, per engine:

- visibility fell by at least the configured threshold, in percentage points
- the engine cited the brand's domain in the previous run and no longer does

The default threshold is 5 points. At the sample sizes this template produces, movement smaller than that is not distinguishable from noise, and the report footer says so.

---

## How much data is enough

Answers are not deterministic. The same question asked twice can return a different set of brands and sources.

| Setup | Unbranded answers per run (29 unbranded questions, 4 engines) |
|---|---|
| 1 repeat (default) | about 116 |
| 3 repeats | about 348 |
| 5 repeats | about 580 |

The template ships with one repeat, because one execution is limited to 30 minutes. That is enough for the average across engines to be steady, but each single engine rests on about 29 answers, so one answer moves its figure by about 3.4 points. Two engines can therefore show identical numbers by coincidence.

Prefer more questions over more repeats: variation between questions is larger than variation within one. Raise the repeats only if the run still fits the time limit.

---

## What is not measured

**Sentiment.** How favourably the brand is described flips far more often than whether it is mentioned at all. Scoring it would add a noisy column that invites over-reading.

**Real query volume.** The prompt set approximates what buyers ask. It is not a sample of real queries and cannot say how often any of them is asked.

**Answer position within a response.** Recorded per answer, relative to competitors, but not shown as a headline number: it only means something when the brand is mentioned at all.
