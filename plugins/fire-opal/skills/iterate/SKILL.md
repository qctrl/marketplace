---
name: iterate
description: Run iterative or batch Fire Opal workloads inside one reusable provider session (iterate, iterate_expectation, stop_iterate). Covers parameter sweeps, more than 300 circuits, and variational loops such as VQE.
argument-hint: "<sweep|vqe|stop> <circuit(s)> [details]"
disable-model-invocation: true
allowed-tools: mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_validate mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_show_supported_devices mcp__plugin_fire-opal_qctrl__qctrl_list_credentials mcp__plugin_fire-opal_qctrl__qctrl_get_job mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_get_circuit_results mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_stop_iterate Bash(sleep:*)
---

# Iterative and batch workloads with Fire Opal

Request: `$ARGUMENTS`

`qctrl_fire_opal_iterate` and `qctrl_fire_opal_iterate_expectation` behave like execute and estimate expectation, with one difference: Fire Opal keeps a provider session open between calls (on IBM, a Qiskit Runtime session). Repeated submissions then skip most of the queue and preprocessing time. They run on IBM Cloud credentials only.

**The session must be released with `qctrl_fire_opal_stop_iterate` when the work is done**, with the same `backend_name`, `credential_name` and `organization`. Call it when the loop ends, fails, or the user stops it. Every path below makes sure that happens. It is safe to call more than once.

Each step is one MCP call, then `qctrl_get_job` until it finishes, then a results call. Between `qctrl_get_job` checks, wait about 10 seconds, longer each time, with a background `sleep` command. Never call `qctrl_get_job` again straight away. Keep the number of steps small: put many circuits or parameter sets in each step (up to 300 per call), and agree `max_steps` with the user. Pass `step` (1, 2, …) and `max_steps` on every call: the step limit applies only when both are given.

**Only step 1 validates its circuits.** Later steps must send the same circuit list as step 1 and change only `parameters`. When the chunks hold different circuits, validate every chunk with `qctrl_fire_opal_validate` (free) before step 1.

## Pick the path that matches the request

### A. Sweep or batch: a fixed set of circuits or parameter values known up front

1. **Get OpenQASM text.** Read `.qasm` files. For a `.py` file, export its Qiskit circuits with the user's own Python, as `/fire-opal:validate` describes.
2. **Plan the chunks.** `parameters` is one set per circuit, in the same order. To run every parameter set on a circuit, repeat the circuit once per set. Split the circuit list into chunks of at most 300. Where you can, build every chunk from the same circuit list and vary only `parameters`.
3. **Validate** (free) with `qctrl_fire_opal_validate`: the first chunk, and every chunk whose circuits differ from the first chunk's. Stop on problems. State the size before you start: total circuits, number of chunks (steps), backend and shots. Ask the user to confirm this plan and wait for a clear yes before step 1. Do not rely on the tool permission prompt: in some permission modes it does not appear.
4. **Run each chunk** with `qctrl_fire_opal_iterate` (counts per bitstring) or, when the user gave observables, `qctrl_fire_opal_iterate_expectation`. Pass `step` (1, 2, …) and `max_steps` (the number of chunks). After each call, check `qctrl_get_job` until it finishes, then read each circuit with `qctrl_fire_opal_get_circuit_results`. Save the results as you go to `fire-opal-results/sweeps/<name>.json`, so a failure keeps the finished chunks.
5. **Release the session** with `qctrl_fire_opal_stop_iterate`, even if a chunk failed.
6. **Report** how many items finished, that the session was released, and the saved file. Summarize the results, for example expectation value vs. parameter, or the top bitstring per item. Offer a plot.

### B. Variational loop: the next parameters depend on the last result (VQE, custom QAOA, etc.)

You are the classical optimizer. There is no local Fire Opal script: each step goes through the MCP tools.

1. Get the ansatz as OpenQASM 3 text with named parameters, and validate it with sample `parameters`.
2. **Agree the plan** with the user: ansatz, observable, optimizer, shots, backend, and `max_steps` (the most steps, 1 to 500). Each step is a hardware job. Wait for the user's clear yes before step 1.
3. **Use few, wide steps.** In each step, evaluate many parameter points at once by repeating the ansatz once per point, up to 300 per call. Send the same circuit list on every step and change only `parameters`. For example, evaluate a whole gradient with the parameter-shift rule, a full SPSA pair, or a population of candidates in one step. This needs far fewer steps than one point per call.
4. **Each step**: call `qctrl_fire_opal_iterate_expectation` with `step` and `max_steps`, check `qctrl_get_job` until it finishes, read `expectation_values` with `qctrl_fire_opal_get_circuit_results`, then compute the next parameters. Use the user's Python (numpy, scipy) for the update when the arithmetic is more than a few numbers.
5. Print progress to the user every few steps, and keep the history (parameters, energy, job ID) in `fire-opal-results/` as JSON.
6. **Stop** when the result converges, `max_steps` is reached (the tool then refuses with `step-limit-reached`; don't raise `max_steps` without the user), or the user says stop. Then call `qctrl_fire_opal_stop_iterate`.
7. Report the final parameters and energy, the convergence history, and any exact or classical reference value the user provides.

Very long loops (hundreds of steps) are slow and costly this way. Tell the user so before you start, and suggest fewer, wider steps.

### C. One step at a time: the user wants to drive iterations manually

First agree `max_steps` with the user, and wait for a clear yes before step 1. Then call `qctrl_fire_opal_iterate` or `qctrl_fire_opal_iterate_expectation` as the user directs, with `step` and `max_steps`, and the same circuits on every step. Report each result. When they're done, or before ending the conversation, call `qctrl_fire_opal_stop_iterate`.

### D. "stop" or "release the session"

Call `qctrl_fire_opal_stop_iterate` with the backend (and credential, if the user has more than one), and confirm. It closes the session for every loop on that backend with that credential.

## IBM run options

All the submit tools accept `run_options` on IBM:

- `job_tags`: tags shown in the IBM Quantum dashboard
- `session_id`: run inside an existing IBM Runtime session
- `reduce_approximation`: keep near-identity gates during compilation
