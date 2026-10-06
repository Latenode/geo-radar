# Customizing

Everything the sheet cannot change is here. The sheet-level settings are in the README; this page covers node settings and code.

Nothing below needs anything beyond editing a node. The code is plain JavaScript with English comments.

---

## Before you edit code

Code refers to other nodes by number, for example `{{$16.metrics}}` or `{{$28.actions}}`. Two consequences:

- Editing a node keeps its number. Deleting a node and adding a new one gives it a new number, and every reference to the old one goes empty without an error.
- If you replace a node in the main chain, either keep its number or search the code of the nodes after it for the old reference and update it.

Nodes read their inputs in two ways, and you will see both in every script. Values come from the node's parameters, and when a parameter is empty the script falls back to reading the value straight from the run data by its substitution key. Leave that pattern alone; it is what makes the values arrive.

Save after each change, then run once on a cheap setup (one engine, one repeat) before trusting it.

---

## Constants

| Node | What | Default | Where in the code |
|---|---|---|---|
| `build_run_matrix` | Upper limit for `runs_per_prompt` | 10 | `Math.min(..., 10)` |
| `build_run_matrix` | Repeats used when the cell is empty | 3 (does not fit a four-engine run in 30 minutes, so always fill the cell) | the fallback value after `parseInt` |
| `compute_deltas` | Alert threshold when the cell is empty | 5 points | the fallback value after `parseFloat` |
| `aggregate_sources` | Length of the accumulation window | 28 days | `28 * 24 * 60 * 60 * 1000` |
| `aggregate_sources` | Share of runs a domain must appear in to count as stable | 0.5 | `persistence >= 0.5` |
| `aggregate_sources` | Domains kept in the stable list | 15 | `.slice(0, 15)`, two places |
| `aggregate_metrics` | Domains kept in `top_sources` and `gap_sources` | 15 | `.slice(0, 15)`, two places |
| `build_report` | Gap sources shown in the email | 8 | `.slice(0, 8)` |
| `build_reco_prompt` | Gap sources sent to the model | 15 | `.slice(0, 15)` |
| `build_reco_prompt` | Actions requested | 4 to 6 | text of rule 7 |
| `unique_prompts` | Category skipped in the Google check | `brand_research` | `cat === "brand_research"` |
| `build_gen_prompt` | Questions written per brand | 32, split 4 / 7 / 9 / 5 / 4 / 3 | the category lines in the instruction |
| `build_gen_prompt` | Homepage text passed to the model | 6000 characters | `.slice(0, 6000)` |

### Notes on the ones people change most

**Window and stability.** A shorter window makes the source list react faster and lets one-off domains in. A higher stability value demands more consistency and leaves fewer domains. With only a few weeks of history, expect the node to report that it fell back to plain volume; that is normal.

**Number of generated questions.** If you change the split, keep the total in the line that says how many to write, and keep three `brand_research` questions. The category names must stay exactly as they are - the metrics group by them.

**Alert threshold.** Below 5 points, movement between runs is mostly noise at these sample sizes. Lowering it makes alerts frequent and mostly meaningless unless you also raise `runs_per_prompt`.

---

## Answer length and run time

These are node settings, not code.

| What | Where | Default |
|---|---|---|
| The length sentence | The prompt field of each engine node (`Prompt` on Perplexity, `User Prompt` on the others), written after the question variable | `Answer in no more than 150 words.` |
| Max Tokens | The same nodes | 400 on Perplexity, OpenAI and Claude; 1000 on Gemini |
| Web search weight | `Web Search Server Context Size` and `Web Search Max Results` on the Claude and Gemini nodes | `low` and `3` |
| Claude model | The Claude node | `claude-haiku-latest` |

Change the sentence in all four nodes together. If the engines get different instructions, the answers stop being comparable.

Keep Max Tokens high on Gemini. It reasons before it answers, and a low limit can leave no room for the answer.

The question text for the Google check comes from a different place, so the length sentence never reaches it. If you add the sentence anywhere upstream of the engines, the Google check will search for it.

Lengthening the answers makes every call slower. Check the time under each engine node after a run, and remember that one execution is limited to 30 minutes.

---

## Changing how recommendations are written

The instruction sent to the model is built in `build_reco_prompt`. It has three parts you can edit:

1. **The data.** Engine table, competitor board, gap sources, alerts. Add lines here to give the model more to work from.
2. **Action types.** Each type is defined by what its target must be. Types that act on the brand's own site require the brand's domain; types that act on outside sites require a third-party domain.
3. **Hard rules.** Numbered, in plain sentences. Add, reword or remove them.

Rules worth keeping, because the model breaks them otherwise:

- competitor-owned domains are never pitch targets
- the year in any title is the current year, passed in as a field
- a roundup is never proposed on the brand's own blog

Keep the JSON shape at the end of the instruction. `parse_recommendations` reads `headline`, `diagnosis` and an `actions` list with `priority`, `type`, `target`, `action` and `evidence`, and `build_report` displays those fields.

If you rename an action type, also update the type check in `build_report`, which relabels an action whose type contradicts its target.

---

## Changing the report

`build_report` builds the email as one HTML string. Sections appear in a fixed order: header, headline number, diagnosis, alerts, Google comparison, per-engine table, competitor board, sources, action cards, footer.

- To remove a section, delete its variable from the concatenation that builds the HTML.
- To change colours and spacing, edit the style constants near the top of the HTML section.
- Wording is in the strings. It is English; translating means editing those strings.
- The Slack text is built in the same node, further down.

Keep the step that writes a hidden element after each dot in a domain. Without it mail clients turn every domain into a link, and this report lists competitors.

---

## Adding an engine

An engine is one branch plus one line in the parser.

1. After `iterate_calls`, add a branch to the router with the condition `{{$6.value.engine}}` equal to the new engine's name.
2. Add an LLM node with web search on, and connect its output to `normalize_response`.
3. In `normalize_response`, add a parameter that points at the new node's output, like the existing ones for the other engines. The script only sees values that a parameter references. Then add the new name to the condition that handles the All LLM Models engines, and add its parameter and node number to the mapping just below it. Nodes of that type share one response shape, so nothing else changes.
4. Add the name to `engines_enabled` for the brands that should use it.

Metrics, the competitor board, recommendations and the report all read engines from the data, so they pick the new one up without changes.

A native node with a different response shape needs its own parsing branch in `normalize_response`, following the Perplexity one.

---

## Switching parts off

**One engine.** Remove it from `engines_enabled`. No node changes.

**A single brand or question.** Set `active` to `FALSE`.

**Slack.** Delete `send_alert` and the router link in front of it. The email is unaffected.

**The email.** Delete `send_email` if you want the sheet and Slack only.

**The Google check.** Delete `unique_prompts`, `serp_iterate`, `serper_search`, `serp_position` and `serp_summary`, and connect `aggregate_sources` directly to `build_reco_prompt`. The report leaves out the Google block when it has no data for it. Run once to confirm. This also removes those search calls from the cost of a run.

**Recommendations.** They cost one model call per brand per run, which is small. Removing them means the report has no action cards; do this only if you want measurement alone.

**The report recipient.** `send_email` reads the address from `config`. A fixed address typed into its To field overrides that for every brand.

---

## Resetting

**Start a fresh trend.** Set a new `prompt_set_version` in `config`. Old rows stay; new runs are compared only with runs carrying the new label.

**Forget the source history.** Delete the brand's rows in `source_log`. The next run rebuilds the window from scratch and will report a fallback until it has a few runs.

**Start completely clean.** Clear everything below the header row in `results_log`, `metrics_history` and `source_log`. Leave the headers.

**Keep the logs small.** `results_log` grows by one row per answer, roughly a thousand rows a week at full size. Archive old rows to another tab or file every few months; nothing reads them back except you.

---

## Testing a change cheaply

Set one engine and one repeat in `config`, and set most questions to `FALSE`. Ten questions on one engine is ten calls and costs cents.

Check, in this order:

| Where | What |
|---|---|
| `build_run_matrix` | `total_calls` is the number you expect |
| `results_log` | rows have `brand_id`, `engine` and `category` filled |
| `aggregate_metrics` | `unbranded_responses` and `branded_responses` add up to the rows written |
| `parse_recommendations` | `parsed_ok` is true and there are 4 to 6 actions |
| The email | it arrives, and every section you expect is there |

Then restore the real settings.
