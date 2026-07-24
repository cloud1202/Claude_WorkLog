# worklog — Claude Code skill

A [Claude Code](https://claude.com/claude-code) skill that reconstructs "what did I actually work on today?" by cross-referencing your local Claude Code session transcripts with your git history. No manual logging required.

## What it does

Ask Claude Code to recap your day, and it will:

1. Find the Claude Code sessions you had on the target date (default: today) under `~/.claude/projects`.
2. Pull out what you actually typed and which files got edited in each session.
3. Cross-reference that with `git log` / `git status` for the repo(s) you were in.
4. Give you a plain-language summary grouped by topic — what got done, which files were touched, and whether it was committed.

## Install

Copy (or clone) the `worklog/` folder into your `~/.claude/skills/` directory:

```bash
git clone https://github.com/<your-username>/claude-worklog-skill.git
cp -r claude-worklog-skill/worklog ~/.claude/skills/worklog
```

Or just download `worklog/SKILL.md` and place it at `~/.claude/skills/worklog/SKILL.md`.

## Usage

From inside any git repo, ask Claude Code things like:

- "What did I work on today?"
- "Summarize yesterday's work"
- "/worklog 2026-07-20"

## How it works

The skill only reads files that already exist on your machine — Claude Code's own local session transcripts (`~/.claude/projects/**/*.jsonl`) and your local git metadata (`git log`, `git status`, `git diff`). Nothing is uploaded or sent anywhere beyond what you already share with Claude Code itself.

## License

MIT — see [LICENSE](LICENSE).
