# AI log

> **DRAFT — edit this before submitting.** This was written alongside the code, but only you know what actually happened in your sessions. Replace the bracketed parts with your real prompts, mistakes you caught, and hand edits. Reviewers read this to see how you work with AI, so honest and specific beats polished.

## Tools used
- Claude (chat) for the initial design and all first-draft code: schema, slot engine, booking service, HTTP layer, UI pages, tests, docs.
- [Add anything else: Cursor / Copilot / etc.]

## Key prompts
1. "Complete the assignment" with the full brief pasted in. [Add the prompts you actually wrote.]
2. [e.g. "Why a host-row lock instead of an exclusion constraint?"]
3. [e.g. "Explain the DST example in my own words"]

## Where the AI got things wrong (found while reviewing)
- **Directory creation:** `mkdir -p {a,b}` failed in `sh` (no brace expansion) and created a literally-named folder; caught from the command output and redone.
- **Tests vs buffers:** the first draft of the booking tests assumed adjacent hourly slots could both be booked. With a 30-min buffer they conflict (correctly). Fixed the tests to use non-adjacent slots.
- **Off-grid test time:** the cross-event overlap test originally used a start time (+75 min) that is not on the other event's 30-min grid, so it would have failed for the wrong reason. Changed to +60 min and made slot selection robust near end of day.
- [Add what YOU caught, e.g. library behaviour, SQL mistakes, anything that did not run.]

## What was checked
- The slot/DST logic was executed and its 9 unit tests pass.
- All TypeScript files and both pages' inline scripts were syntax-checked.
- **Not executed by the AI:** the Postgres-backed tests and the running app (no database or network in its sandbox). [Replace with your own run results.]

## What I changed by hand
- [List your edits: naming, UI tweaks, bugs fixed after running it, decisions you disagreed with.]
