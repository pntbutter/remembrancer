# Boost: Regenerate After Editing Guidelines

`php artisan boost:update --no-discover --no-interaction` is allowed and expected, because its only output is the AI markdown files (`CLAUDE.md`, `AGENTS.md`, per-agent skill dirs) which agents may write.

Always run it immediately after changing anything in `.ai/guidelines/**`, otherwise the change is invisible — the composed `CLAUDE.md` is what actually enters context. Use `--no-discover` to keep the run non-interactive and avoid silently adopting newly published guidelines; drop the flag only when the human explicitly wants to review new ones.
