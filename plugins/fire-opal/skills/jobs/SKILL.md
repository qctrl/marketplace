---
name: jobs
description: List recent Fire Opal jobs with their status and job IDs. Use when the user asks about their Fire Opal job history, queued jobs, or what they ran recently.
argument-hint: "[limit] [SUCCESS|FAILURE|REVOKED|PENDING|RECEIVED|RETRY|STARTED]"
---

# Fire Opal job history

Arguments: `$ARGUMENTS` — an optional number of jobs to show (default 10) and an optional status filter.

Call `qctrl_list_jobs` with `product` `fire-opal`, `limit`, and `status` when given. If `has_more` is true and the user wants more, call it again with `offset` set to `next_offset`.

Show the jobs as a table: job ID, workflow, status, progress, and created time (UTC). Point out anything still `PENDING`, `RECEIVED`, `STARTED` or `RETRY`, and say that `/fire-opal:result <job_id>` fetches a finished job's result. If the call fails because the user is not signed in, tell them to run `/mcp` and sign in to `plugin:fire-opal:qctrl`.
