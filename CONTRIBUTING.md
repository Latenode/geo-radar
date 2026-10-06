# Contributing

Thanks for considering a contribution.

## What is useful

- **Prompt sets for other categories.** The generator writes a first draft from a homepage; a hand-tuned set for a specific market is worth sharing.
- **Engine adapters.** A new engine needs one branch and one response parser in `normalize_response`.
- **Report layouts.** The HTML email is built in one node.
- **Bug reports.** Say which engine, what the node returned, and what you expected.

## Never commit secrets

Scenario exports and screenshots leak easily. Before opening a pull request, check that you have not included:

- connection identifiers (`{{#...}}` values in node parameters)
- API keys or bearer tokens
- spreadsheet IDs
- workspace, folder or owner identifiers
- email addresses
- real client names, domains or traffic figures in screenshots

Placeholders in the export look like `YOUR_SPREADSHEET_ID` and `{{#YOUR_GOOGLE_SHEETS_CONNECTION}}`. Leave them as placeholders.

## Design rules

- **Keep visibility and brand knowledge separate.** Never average branded and unbranded answers into one number.
- **Fix the parameters the model must not choose.** Anything a calculation depends on stays fixed in the node.
- **Cap every loop.** Calls to external services cost money and hit rate limits.
- **Compare only within a prompt set version.**
- **No comments or strings in code in any language other than English.**
- **Do not mark an action as a pitch target when the domain belongs to a competitor.**
