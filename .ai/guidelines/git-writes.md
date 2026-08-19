# Git: Read Freely, Never Write

Read-only git is always fine: `status`, `diff`, `log`, `show`, `blame`, `ls-files`, `branch --list`, `gh pr view`.

Never run a git command that changes repository or index state unless the human asks for that exact action in that message: `commit`, `add`/`stage`, `rm`, `restore`, `checkout`, `switch`, `branch`, `merge`, `rebase`, `reset`, `stash`, `tag`, `push`, `pull`, `fetch`, `gh pr create`.

Blanket approval does not carry across turns — "commit that" authorizes one commit, not the next one.

## Do Not Suggest Git Commands Either

Never end a response with `git add ...`, a commit reminder, or a staging suggestion. The human runs their own version control on their own schedule and does not need prompting. Unprompted git advice is noise.

Only bring up git when asked to. When asked for a commit message, write the message — do not also print the command to run it.
