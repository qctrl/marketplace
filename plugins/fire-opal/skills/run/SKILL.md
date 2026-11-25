---
name: run
description: Validate and run quantum circuits on real hardware through Fire Opal's error-suppression pipeline, then retrieve and summarize the results.
argument-hint: "<circuit.qasm|circuits_dir|circuit.py> [backend] [shots]"
disable-model-invocation: true
allowed-tools: mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_validate mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_show_supported_devices mcp__plugin_fire-opal_qctrl__qctrl_list_credentials mcp__plugin_fire-opal_qctrl__qctrl_get_job mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_get_circuit_results Bash(sleep:*)
---

# Run circuits with Fire Opal

Arguments: `$ARGUMENTS` — circuit source, then optional backend name, then optional shot count. If no circuit source was given, ask for one. If no backend was given, call `qctrl_fire_opal_show_supported_devices` and ask. If no shot count was given, suggest 2048.

Submitting uses the user's hardware-provider time and Fire Opal usage. Never submit without the user's clear yes in the chat.

1. **Get OpenQASM text.** Read `.qasm` files. For a `.py` file, export its Qiskit circuits with the user's own Python (`qiskit.qasm2.dumps`, or `qiskit.qasm3.dumps` for circuits with unbound parameters), as `/fire-opal:validate` describes. Stop and explain if any circuit has no measurements.

2. **Validate first** (free): call `qctrl_fire_opal_validate` with the circuits, `backend_name` and any `parameters`. If it reports problems, stop, explain them and offer fixes. Pass on warnings but continue.

3. **Submit.** State in one line what will run: number of circuits, backend, shot count. Then ask the user to confirm and wait for a clear yes. Do not rely on the tool permission prompt: in some permission modes it does not appear. Then call `qctrl_fire_opal_execute`.

   - `parameters` is only for parametric circuits: one set per circuit, in the same order. Amazon Braket credentials do not accept parameters.
   - At most 300 circuits per job. For more, or for many parameter values, use `/fire-opal:iterate`.
   - On IBM, pass `run_options` when the user asks: `job_tags` (shown in the IBM Quantum dashboard), `session_id` (an existing Runtime session) or `reduce_approximation` (keep near-identity gates, for circuits where tiny rotations matter).

   Keep the `job_id`.

4. **Wait for the result.** Call `qctrl_get_job` until the status is `SUCCESS`, `FAILURE` or `REVOKED`. Between checks, wait about 10 seconds, longer each time, with a background `sleep` command. Never call `qctrl_get_job` again straight away. Hardware queues can take minutes to hours: after a few checks, tell the user the job is queued, give the job ID, and say `/fire-opal:result <job_id>` will fetch it later.

5. **Summarize** on success. Call `qctrl_fire_opal_get_circuit_results` once per circuit (`circuit_id` 0 to the number of circuits minus 1):

   - The most likely bitstrings with their probabilities. Fire Opal returns post-processed probabilities, not raw counts.
   - If the user stated an expected answer (for example a hidden bitstring), report its probability.
   - Results should be taken from Fire Opal rather than the provider's console, which lacks Fire Opal's post-processing.

   On `FAILURE` or `REVOKED`, show the errors from `qctrl_get_job`.

Offer to plot the distribution if it would help. For the full data, `/fire-opal:result <job_id>` saves the results file.
