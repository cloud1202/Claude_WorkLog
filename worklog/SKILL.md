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

- The repo may be shared with teammates, so always scope commits to the user themselves: get their identity with `git config user.email` (fall back to `user.name`) and add `--author="<identity>"`.
  `git log --author="<user.email>" --since="<date> 00:00" --until="<date> 23:59:59" --date=local --name-status --pretty=format:'%h|%ad|%s'`
- Never report unfiltered `git log` output as "what the user did" — without the author filter, teammates' commits on the same day get attributed to the user, producing a wrong summary.
- If the target date is today, also check uncommitted work: `git status --porcelain` and `git diff --stat` (this is local working-tree state, so no author filtering is needed there).
- Repeat per repo if multiple repos are in scope.

## 4. Report

Match session-touched files against git-changed files by path, then summarize grouped by topic/feature — not by session count:

- Target date(s) and repo(s)
- For each topic: a sentence or two on what was done, the files involved, and whether it was committed (hash + message) or still uncommitted
- Collapse multiple sessions on the same topic into one narrative — the goal is "what got done," not "how many sessions it took"
- Mention discussion-only sessions (no file changes) briefly, only if relevant

## 5. Visualize workload

Give the user a relative sense of how big each topic was.

- **Primary sizing metric**: files changed and diff lines changed (insertions + deletions).
  - For the user's own commits (from step 3): `git show --stat <hash>`.
  - For tracked-but-uncommitted files: `git diff --stat -- <files>`.
  - For new (untracked) files, diff won't show them — count the whole file as added lines with `wc -l <file>`.
  - Sum across every commit/file that belongs to the same topic.
- **Reference metrics (do not fold into the primary size)**: for the session(s) behind that topic, the span between the first and last human-authored message timestamp on the target date (duration), and the sum of `message.usage.output_tokens` across assistant turns on that date (token usage). Filter to the target date only if the session spans multiple days. Keep these separate from the ranking — token count and duration correlate weakly with actual complexity.
- **Load the `dataviz` skill before building any chart** and follow its procedure (form → color → marks → accessibility). This case is a single metric (diff lines) compared across topics, so the form is a horizontal bar chart, one series (no legend needed), colored with the palette's default sequential hue (blue) only.
- Bar length = diff lines, with a value label at the tip. Put files-changed count and commit status (committed/uncommitted) as secondary text on the row; put duration and token usage in a hover tooltip and a table-view toggle.
- Publish it as an Artifact (write the file to the scratchpad directory, then publish). Keep the chat reply to a short summary of the key numbers plus the artifact link.
- Define the theme's CSS custom properties (`--text-primary`, etc.) on `:root`, not on an inner wrapper class like `.viz-root`. Scoping them to the wrapper only means any descendant that doesn't set its own color (e.g. an `h1`) falls back through inheritance to `body`, hits an undefined var there, and keeps whatever ambient color it inherited — in dark mode this leaves dark-on-light text sitting on a dark card, i.e. invisible text. Double-check every text element has an explicit color, not just an inherited one.
- If the user asks for a circular (donut/pie) form, offer it **in addition to** the bar, not instead of it. Keep the bar for precise magnitude comparison; use the donut only for "what share of today's total does this topic account for" (part-to-whole), with ≤6 segments, and skip it if the values are too close to distinguish by angle (bar or plain numbers instead). Because each topic is now an identity (not a ranked magnitude), use the categorical palette (fixed slot order 1, 2, 3, …) instead of the single sequential hue, and run `validate_palette.js` even for 3 slots. Always pair it with a legend (swatch + name + value + share%) so identity never depends on color alone.

## Notes

- Session transcripts contain a lot of noise (tool output, system reminders). Only treat human-typed text and actual file-editing tool calls as signal.
- Timestamps inside the JSONL are UTC (`Z` suffix); file modification times are shown in local time. Use file mtime for day-boundary filtering, and only fall back to converting timestamps near midnight if results look ambiguous.
- In a shared repo, never report raw `git log` output as the user's own work without the author filter (see step 3) — that misattributes teammates' commits.
- This skill only reads data that already exists locally (Claude Code's own session logs and local git metadata) — it doesn't call out to any external service.
