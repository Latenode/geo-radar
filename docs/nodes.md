# Node reference

What each node in the scenario does, and which values are locked and why.

The scenario has two independent branches. The main branch starts at the Schedule Trigger. The prompt generator starts at `gen_trigger` and is run by hand.

---

## Main branch

### Reading and planning

**read_config** - Google Sheets, Search Rows. Returns every row of `config` where `active` is TRUE.

**read_prompts** - Google Sheets, Search Rows (Advanced). Returns every row of `prompts` where `active` is TRUE. The Google Sheets nodes return columns as letters (`A`, `B`, `C`), not as header names. The next node maps them back.

**build_run_matrix** - JavaScript. Expands brands x prompts x engines x repeats into one flat array. A flat array means a single loop instead of three nested ones, which keeps the canvas readable.

Each item carries its brand's settings with it (aliases, domain, competitors). Nodes inside the loop read those from the iterator, never from `read_config`, because the loop interleaves brands.

The matrix is ordered by repeat: repeat 1 of every question and engine comes before any repeat 2, so a run that is cut short still has asked every question once. The result also carries `started_at`, the moment the run began. No node reads it at the moment.

Returns a `total_calls` count. Check it before every large run.

### Asking the engines

**reset_accumulator** - JavaScript. Creates an empty accumulator in workspace storage. The key includes the execution id, so two overlapping runs never mix data.

**iterate_calls** - Iterator over the run matrix. Two outputs: the loop body, and the exit that fires after the last item.

A router after the iterator sends each call to the engine named in the item:

| Node | Engine | Notes |
|---|---|---|
| `engine_perplexity` | Perplexity | Native node. Always searches the web and always returns citations. |
| `engine_openai` | ChatGPT | All LLM Models node with web search enabled. |
| `engine_claude` | Claude | All LLM Models node with web search enabled. |
| `engine_gemini` | Gemini | All LLM Models node with web search enabled. |

Each engine node appends one sentence to the question: `Answer in no more than 150 words.` The other settings that keep a run inside the time limit are listed in the README, under "Staying inside the time limit".

Max Tokens is 400 on the Perplexity, OpenAI and Claude nodes and stays at 1000 on the Gemini node. Gemini 2.5 Pro reasons before it answers, spends part of the limit on that, and can return an empty answer if the limit is low.

**normalize_response** - JavaScript. Brings the four response formats to one structure, then:

- extracts the answer text and the cited URLs
- reduces URLs to domains and removes duplicates
- detects the brand and each competitor with a word-boundary match
- records the brand's position relative to competitors, by order of first appearance

Matching is done on word boundaries so that a brand name that is also an ordinary word does not fire on every sentence.

**accumulate** - JavaScript. Appends the normalized result to the accumulator, together with brand id, prompt id, category and the prompt set version. 

### After the loop

**prepare_rows** and **write_raw_log** - Turn the accumulator into an array of arrays in the exact column order of `results_log` and append it. The Google Sheets "Add Multiple Rows" node takes positional arrays, so column order is fixed by the sheet, not by field names.

**group_by_brand** - JavaScript. Splits the accumulator by brand and attaches each brand's settings.

**iterate_brands** - Iterator over brands. Everything from `aggregate_metrics` to `send_alert` runs once per brand, with that brand's Slack channel and email recipient.

### Metrics

**aggregate_metrics** - JavaScript. Per engine and per brand:

- `visibility` - mention rate on unbranded prompts only
- `brand_knowledge` - mention rate on `brand_research` prompts only
- `citation_rate`, `share_of_voice`, `avg_position`

Also builds:

- `competitor_board` - mentions per competitor and for the brand, on unbranded answers
- `gap_sources` - domains cited in answers where at least one competitor appears and the brand does not
- `top_sources` - all cited domains with counts

**read_last_metrics** - Reads the brand's history from `metrics_history`.

**compute_deltas** - JavaScript. Compares this run to the most recent earlier run for the same engine, and only within the same `prompt_set_version`. Fires an alert when visibility falls by more than the threshold, or when an engine that used to cite the brand's domain stops doing so. Ignores rows that belong to the current run and rows with a malformed run id.

**prepare_metrics_rows** and **write_metrics** - Positional array in the column order of `metrics_history`, then append.

### Source accumulation

**prepare_source_rows**, **write_source_log** - Write every cited domain of the run to `source_log`, flagged when it is a gap source.

**read_source_history**, **aggregate_sources** - Read the brand's history and aggregate over a 28-day window. For each domain: total answers, number of runs it appeared in, and persistence (the share of runs in the window that saw it).

A domain counts as stable when it appeared in at least half of the runs in the window. With a short history there is nothing stable yet, so the node falls back to plain volume and sets `used_fallback`.

### Classic Google check

**unique_prompts** - Reduces the accumulator to one entry per unbranded prompt. Branded prompts are skipped, since asking Google about your own name proves nothing.

**serp_iterate**, **serper_search**, **serp_position** - For each prompt, one Serper search. Records the brand's position among the top results and each competitor's.

**serp_summary** - Collects the results and deletes its own accumulator. Returns the share of prompts where the brand reaches the top 10 and the average position.

The Google check runs before the recommendation step so that its result is available to the report.

### Recommendations and report

**build_reco_prompt** - JavaScript. Builds the instruction for the model from the measured data. Three things in it matter:

1. Gap domains that belong to a listed competitor are tagged in the data as do-not-pitch. Left unmarked, the model proposes contacting the competitor itself.
2. Each action type is defined by what its target must be. Types that act on the brand's own site require the brand's domain as target; types that act on outside sites require a third-party domain.
3. The current year is passed in as a field and stated as a rule. Without it the model writes the year from memory.

**recommend** - LLM node, no web search, empty Max tokens, low temperature.

**parse_recommendations** - JavaScript. Latenode parses the model's JSON itself and places the object in `content`. The node reads that first and only falls back to text parsing if it is a string.

**build_report** - JavaScript. Builds the HTML email and a short Slack text. Two details worth knowing:

- Domains are written with a hidden element after each dot. Mail clients otherwise turn every domain into a clickable link, which in this report would send readers to competitors.
- An action whose stated type contradicts its target gets a neutral label instead of the wrong one.

**send_email** - Gmail. Runs every time. The To field holds `{{$15.value.email_to}}`, so each brand's report goes to the address in `config`. Body type is HTML.

**send_alert** - Slack. Sits behind a router that passes only when `has_alerts` is true, so a quiet week produces no message.

**cleanup** - Runs after the last brand and deletes the accumulator.

---

## Prompt generator branch

Run by hand, once per brand.

| Node | Role |
|---|---|
| `gen_trigger` | Manual trigger |
| `gen_read_brands` | Reads active brands from `config` |
| `gen_iterate_brands` | One pass per brand |
| `fetch_site` | GET request for the brand homepage |
| `build_gen_prompt` | Strips scripts, styles and tags, keeps the first 6000 characters, and builds the instruction |
| `generate_prompts` | LLM node, no web search, higher temperature for varied phrasing |
| `parse_prompts` | Validates categories, removes duplicates, stamps version and date |
| `write_prompts` | Appends the rows to `prompts` |

The instruction asks for 32 questions across six categories and enforces one hard rule: only the three `brand_research` questions may contain the brand name. A question that names the brand cannot measure whether engines recall it unprompted.

Generated rows are appended, not written over. Clear the brand's old rows first, or the next run will use both sets.

---

## What is locked and why

| Locked | Reason |
|---|---|
| Aggregation groups `brand_research` apart from everything else | Mixing them pins every engine at the same number |
| One repeat by default | One execution is limited to 30 minutes; coverage comes from the number of questions instead (see the README) |
| Comparison only inside one `prompt_set_version` | A different question set is a different measurement |
| Citation window of 28 days | Shorter windows let one-off domains dominate the list |
| Alert threshold defaults to 5 pp | Below that, week-to-week movement is noise at these sample sizes |
| Read-only towards every external service | Nothing here writes to a client's systems except the report email and the optional Slack message |
