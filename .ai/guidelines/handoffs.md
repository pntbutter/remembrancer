# Handoffs Are Complete Code, Not Hints

Do not withhold code to force the human to work it out. Typing the code *is* their review pass — they catch what is wrong while typing it, so the block must be complete and directly typeable. No pseudocode, no "fill in the rest", no deliberate gaps as a teaching device.

## Shape Of A Handoff

1. State what changes and why, in a line or two.
2. Give the full file path, then the exact code as a fenced block with enough surrounding context to place it.
3. For edits to existing files, show a unified diff or before/after — never a full rewritten file when only a few lines change.
4. List the follow-up commands to run: `php artisan make:*`, `migrate`, `npm run build`, the specific test filter.
5. Offer to review the result once it is typed.

## What Makes The Typing Pass Productive

- **Deliver in small steps.** Roughly one logical change per handoff. Multi-file work goes out as step 1, then wait — objections should surface after fifteen lines, not after two hundred.
- **State the tradeoff.** Name what you did not pick and why. That is the material they judge your code against while typing it.
- **Flag your own uncertainty inline.** Mark the opinionated or shaky parts explicitly ("this assumes the guard is `web`", "a Form Request would also work here"). Point at where to push back.
- **Treat pushback as signal, not friction.** When they say it is wrong, take it seriously and reason it through with them. Do not reflexively defend the original, and do not reflexively capitulate either.
- **Verify before showing.** Confirm APIs against `search-docs` and the installed versions rather than memory, and name the source. Wrong code typed by hand costs far more than wrong code applied by a tool.
