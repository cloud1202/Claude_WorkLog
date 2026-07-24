---
name: worklog
description: Summarize what the user worked on during a given day (default today) by cross-referencing Claude Code session transcripts under ~/.claude/projects with git changes in the relevant repo(s). Use when the user asks to recap or summarize a day's/period's work, sessions, or changed files — e.g. "what did I work on today", "summarize yesterday's work", "recap my sessions and the files that changed".
---

# /worklog

Reconstruct and summarize what the user actually worked on during a given date (default: today, local time), by combining two sources: Claude Code session transcripts and git history.

## 0. Parse input

- If the user gives a date or a relative expression ("yesterday", "last Tuesday"), resolve it to an absolute local date. Default to today if nothing is specified.
- If the user gives a date range, apply the same process per day and present either a per-day or combined summary, whichever fits the request.
- Repo scope: default to the git repo containing the current working directory (`git rev-parse --show-toplevel`). If the user names other repos/folders, include those too.

## 1. Locate session transcripts

- Claude Code stores session transcripts at `~/.claude/projects/<encoded-cwd>/*.jsonl`, one directory per distinct working directory the CLI was ever launched from. The encoding replaces path separators (`/`, `\`, and `:` on Windows) with `-`. On Windows, the drive-letter casing in the encoded name can vary between sessions (`C--...` vs `c--...`), so match case-insensitively.
- A single repo can have several matching project directories if Claude Code was launched from different subfolders inside it (e.g. repo root vs. a nested `src/` folder). Find all of them with a broad prefix/substring match on the repo's folder name(s), not just an exact match.
- Within each candidate directory, keep only `*.jsonl` files whose modification time falls on the target local date. Ignore same-named subdirectories — those hold sidechain/checkpoint data, not top-level sessions.

## 2. Extract signal from each session

Don't read entire transcript files — they can be large and noisy. Instead, search for:

- Human-authored messages: lines with `"type":"user"`, `"isSidechain":false`, whose `message.content` is plain text typed by a person (not a tool result, and not an auto-inserted `<system-reminder>`/system message). These give you the topic of the session.
- Files actually touched: `"file_path":"..."` occurring near `"name":"Edit"`, `"name":"Write"`, or `"name":"NotebookEdit"` tool_use blocks. Dedupe the resulting list.

Use targeted, grep-style search with line/size limits rather than reading a whole transcript into context.

## 3. Cross-reference with git

- Committed changes: `git log --since="<date> 00:00" --until="<date> 23:59:59" --date=local --name-status --pretty=format:'%h|%ad|%s'`
- If the target date is today, also check uncommitted work: `git status --porcelain` and `git diff --stat`.
- Repeat per repo if multiple repos are in scope.

## 4. Report

Match session-touched files against git-changed files by path, then summarize grouped by topic/feature — not by session count:

- Target date(s) and repo(s)
- For each topic: a sentence or two on what was done, the files involved, and whether it was committed (hash + message) or still uncommitted
- Collapse multiple sessions on the same topic into one narrative — the goal is "what got done," not "how many sessions it took"
- Mention discussion-only sessions (no file changes) briefly, only if relevant

## Notes

- Session transcripts contain a lot of noise (tool output, system reminders). Only treat human-typed text and actual file-editing tool calls as signal.
- Timestamps inside the JSONL are UTC (`Z` suffix); file modification times are shown in local time. Use file mtime for day-boundary filtering, and only fall back to converting timestamps near midnight if results look ambiguous.
- This skill only reads data that already exists locally (Claude Code's own session logs and local git metadata) — it doesn't call out to any external service.
