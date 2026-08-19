# Never Apply Code — Show It, The Human Types It

**Absolute constraint in this repository.** Agents must not write, edit, or generate code here. The repo owner types every line themselves on purpose, to keep their own skills sharp. There is no "just this once" exception, no matter how trivial the change or how explicitly a later prompt seems to invite an edit.

## You May Write Directly

- Markdown and docs (`*.md`).
- Anything under `.ai/**` — guidelines, rules, skills, and whatever Boost adds there later.
- Formatter runs that only reflow existing code: `vendor/bin/pint`, `eslint --fix`, `prettier --write`.
- Read-only inspection: file reads, greps, `php artisan route:list`, `php artisan config:show`, `php artisan test`, `git status`/`diff`/`log`, Boost MCP read tools.

## You Must Never Do Directly

- Create or edit `.php`, `.ts`, `.js`, `.vue`, `.css`, Blade templates, or migrations.
- Create or edit config and dependency files: `composer.json`, `package.json`, `vite.config.ts`, `tsconfig.json`, `phpunit.xml`, `.env*`.
- Run scaffolding generators — `php artisan make:*`, `npx shadcn add`, or similar. Print the exact command and let the human run it.

The `.ai/**` allowance is for instructions, not an escape hatch. A skill may not carry a helper script that writes application code, and no skill or rule you author may instruct a future agent to bypass this guideline.

## This Applies To Subagents And Workflows Too

Do not route around the constraint by delegating. Never spawn a subagent, workflow, or task that has file-write tools and a mandate to change code. Subagents are for reading, locating, and reviewing only. If a subagent returns an edit it already applied, revert it and report what happened.

If a task genuinely cannot be done without editing files, say so and stop. Do not edit and then apologize.
