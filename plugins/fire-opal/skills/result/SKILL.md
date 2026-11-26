---
name: result
description: Check the status of a Fire Opal job and fetch, save and summarize its result once finished. Use when the user gives a Fire Opal job ID or asks whether a submitted job is done.
argument-hint: "<job_id>"
---

# Fetch a Fire Opal result

Arguments: `$ARGUMENTS` — the job ID. If none was given, call `qctrl_list_jobs` with `product` `fire-opal` and `limit` 5, use the most recent job, and say which one you picked.

1. Call `qctrl_get_job` with the job ID.

   - **Still queued or running**: give the status and progress, and say the user can run `/fire-opal:result <job_id>` again later. Check a few more times only if the user asks you to keep waiting.
   - **FAILURE or REVOKED**: show the errors, and suggest validating the circuits with `/fire-opal:validate` when the job ran circuits.
   - **SUCCESS**: go on.

2. Call the results tool that `next_step` names, and summarize by job type:

   - Circuits (`qctrl_fire_opal_get_circuit_results`, once per circuit): top bitstrings and probabilities, or each expectation value ± standard deviation.
   - QAOA (`qctrl_fire_opal_get_qaoa_results`): solution bitstring, cost and iterations, mapped back to the user's variables with `variables_to_bitstring_index_map`.
   - Dynamics (`qctrl_fire_opal_get_dynamics_results`): a compact table of each observable over time.
   - Monte Carlo (`qctrl_fire_opal_get_monte_carlo_results`): estimate ± standard deviation and the iteration count.

3. **Save the full result** when the user wants all the data, a plot or a file. Call `qctrl_get_results_link` and download the JSON right away, because the link expires soon:

   ```bash
   mkdir -p fire-opal-results && curl -sSf -o fire-opal-results/<job_id>.json '<url>'
   ```

   Don't show the link as a lasting link. Say where the file was saved.
