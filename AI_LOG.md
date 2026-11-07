# AI log

## Tools used
- Claude (chat) for the design and the first draft of all code: schema, slot engine, booking service, HTTP layer, the two UI pages, tests, README and report.
- I worked through setup and debugging in the same chat: Docker, GitHub upload, and running the app.

## How I worked
1. I pasted the full brief and asked Claude to build the assessment. It produced the repo and a report.
2. I ran the app locally with Docker Compose and checked that the server started and the database migrations and seed ran.
3. I asked Claude for step-by-step help with each blocker as it came up.

## Where it went wrong or needed fixing
- **Directory creation:** the first scaffold command used `mkdir -p {a,b}`, which does not expand in `sh`, so it created a literally named folder. Claude noticed from the output and redid it.
- **Tests vs buffers:** the first draft of the booking tests assumed adjacent hourly slots could both be booked. With a 30-minute buffer they correctly conflict, so the tests were changed to use non-adjacent slots.
- **Off-grid test time:** one overlap test used a start time that was not on the other event type's 30-minute grid, so it would have failed for the wrong reason. It was changed to start exactly when the first booking ends.
- **Docker on Windows:** Docker Desktop needed the Virtual Machine Platform feature and WSL. The WSL download timed out at 95%, and I worked around it by retrying and installing without a Linux distribution.
- **GitHub upload:** uploading through the web page committed only the top-level files, and the `src`, `migrations`, `public` and `test` folders were missing. I re-uploaded each folder separately.

## What was checked
- The slot and DST logic ran in a sandbox, and its 9 unit tests passed.
- All TypeScript files and both pages' scripts were syntax-checked.
- The app started under Docker Compose with the database healthy, migrations applied and the demo data seeded.
- The Postgres-backed test results are in the report, section 8.

## What I changed by hand
- `<ADD YOUR OWN EDITS HERE: anything you fixed, renamed or decided yourself after running the app>`
