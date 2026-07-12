---
description: Load the most recent session handoff for the current repo — preferring the durable ~/fundswap_github/knowledge-wiki/plans/active/ store, falling back to local ~/.claude/session-data/ — and resume work with full context from where the last session ended.
---

# Resume Session Command

Load the last saved session state and orient fully before doing any work.
This command is the counterpart to `/save-session`.

## When to Use

- Starting a new session to continue work from a previous day
- After starting a fresh session due to context limits
- When handing off a session file from another source (just provide the file path)
- Any time you have a session file and want Claude to fully absorb it before proceeding

## Usage

```
/resume-session                                                      # loads most recent file in ~/.claude/session-data/
/resume-session 2024-01-15                                           # loads most recent session for that date
/resume-session ~/.claude/session-data/2024-01-15-abc123de-session.tmp  # loads a current short-id session file
/resume-session ~/.claude/sessions/2024-01-15-session.tmp               # loads a specific legacy-format file
```

## Process

### Step 1: Find the session file

Two stores exist. **Knowledge-wiki plans is the durable, cross-machine, git-tracked store and is preferred whenever a match exists.** `~/.claude/session-data/` is a local-only scratch store (files are `.tmp`, gitignored if inside a repo, and machine-specific) — treat it as a fallback, not the default source of truth.

If no argument provided:

1. **Check knowledge-wiki plans first**, if `~/fundswap_github/knowledge-wiki/plans/active/` exists on this machine:
   - Determine the current repo name from the working directory (git root basename, or nearest project folder name)
   - Look for files matching `*_<repo-name>_*.md` in that folder
   - If one or more match, pick the most recently modified — this is the session file, skip step 2
2. **Fall back to `~/.claude/session-data/`** if step 1 found nothing (no knowledge-wiki checkout on this machine, or no file tagged for this repo):
   - Pick the most recently modified `*-session.tmp` file
3. If neither store has a matching file, tell the user:
   ```
   No session files found in ~/fundswap_github/knowledge-wiki/plans/active/ (repo: <name>) or ~/.claude/session-data/
   Run /save-session at the end of a session to create one.
   ```
   Then stop.

If an argument is provided:

- If it looks like a file path (absolute, or contains `/`), read that file directly regardless of which store it's in
- If it looks like a date (`YYYY-MM-DD`), search knowledge-wiki `plans/active/` (files starting with that date) first, then `~/.claude/session-data/`, then the legacy `~/.claude/sessions/`, for a matching file and load the most recently modified variant for that date
- If not found, report clearly and stop

**Note on format**: knowledge-wiki plan files use a different section structure (frontmatter `title/date/repo/status`, then freeform headings like 為什麼做/已確認可行/沒有成功的做法/待解決的問題/精確的下一步) than the legacy `-session.tmp` template (WHAT WORKED / WHAT DID NOT WORK / etc). Both map to the same underlying categories — synthesize the Step 3 briefing from whichever structure the loaded file actually uses; do not require the exact English headers to be present.

### Step 2: Read the entire session file

Read the complete file. Do not summarize yet.

### Step 3: Confirm understanding

Respond with a structured briefing in this exact format:

```
SESSION LOADED: [actual resolved path to the file]
════════════════════════════════════════════════

PROJECT: [project name / topic from file]

WHAT WE'RE BUILDING:
[2-3 sentence summary in your own words]

CURRENT STATE:
PASS: Working: [count] items confirmed
 In Progress: [list files that are in progress]
 Not Started: [list planned but untouched]

WHAT NOT TO RETRY:
[list every failed approach with its reason — this is critical]

OPEN QUESTIONS / BLOCKERS:
[list any blockers or unanswered questions]

NEXT STEP:
[exact next step if defined in the file]
[if not defined: "No next step defined — recommend reviewing 'What Has NOT Been Tried Yet' together before starting"]

════════════════════════════════════════════════
Ready to continue. What would you like to do?
```

### Step 4: Wait for the user

Do NOT start working automatically. Do NOT touch any files. Wait for the user to say what to do next.

If the next step is clearly defined in the session file and the user says "continue" or "yes" or similar — proceed with that exact next step.

If no next step is defined — ask the user where to start, and optionally suggest an approach from the "What Has NOT Been Tried Yet" section.

---

## Edge Cases

**Multiple sessions for the same date** (`2024-01-15-session.tmp`, `2024-01-15-abc123de-session.tmp`):
Load the most recently modified matching file for that date, regardless of whether it uses the legacy no-id format or the current short-id format.

**Session file references files that no longer exist:**
Note this during the briefing — "WARNING: `path/to/file.ts` referenced in session but not found on disk."

**Session file is from more than 7 days ago:**
Note the gap — "WARNING: This session is from N days ago (threshold: 7 days). Things may have changed." — then proceed normally.

**User provides a file path directly (e.g., forwarded from a teammate):**
Read it and follow the same briefing process — the format is the same regardless of source.

**Session file is empty or malformed:**
Report: "Session file found but appears empty or unreadable. You may need to create a new one with /save-session."

---

## Example Output

```
SESSION LOADED: /Users/you/.claude/session-data/2024-01-15-abc123de-session.tmp
════════════════════════════════════════════════

PROJECT: my-app — JWT Authentication

WHAT WE'RE BUILDING:
User authentication with JWT tokens stored in httpOnly cookies.
Register and login endpoints are partially done. Route protection
via middleware hasn't been started yet.

CURRENT STATE:
PASS: Working: 3 items (register endpoint, JWT generation, password hashing)
 In Progress: app/api/auth/login/route.ts (token works, cookie not set yet)
 Not Started: middleware.ts, app/login/page.tsx

WHAT NOT TO RETRY:
FAIL: Next-Auth — conflicts with custom Prisma adapter, threw adapter error on every request
FAIL: localStorage for JWT — causes SSR hydration mismatch, incompatible with Next.js

OPEN QUESTIONS / BLOCKERS:
- Does cookies().set() work inside a Route Handler or only Server Actions?

NEXT STEP:
In app/api/auth/login/route.ts — set the JWT as an httpOnly cookie using
cookies().set('token', jwt, { httpOnly: true, secure: true, sameSite: 'strict' })
then test with Postman for a Set-Cookie header in the response.

════════════════════════════════════════════════
Ready to continue. What would you like to do?
```

---

## Notes

- Never modify the session file when loading it — it's a read-only historical record
- The briefing format is fixed — do not skip sections even if they are empty
- "What Not To Retry" must always be shown, even if it just says "None" — it's too important to miss
- After resuming, the user may want to run `/save-session` again at the end of the new session to create a new dated file
