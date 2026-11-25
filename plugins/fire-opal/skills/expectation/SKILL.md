---
name: expectation
description: Estimate expectation values of Pauli observables for quantum circuits on real hardware with Fire Opal's error suppression and mitigation.
argument-hint: "<circuit.qasm|circuit.py> <observable(s)> [backend] [shots]"
disable-model-invocation: true
allowed-tools: mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_validate mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_show_supported_devices mcp__plugin_fire-opal_qctrl__qctrl_list_credentials mcp__plugin_fire-opal_qctrl__qctrl_get_job mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_get_circuit_results Bash(sleep:*)
---

# Estimate expectation values with Fire Opal

Arguments: `$ARGUMENTS` — circuit source, the observable(s), and optionally a backend and shot count. If the observable is missing or ambiguous, ask. This runs on IBM Cloud credentials only.

**Observables.** `qctrl_fire_opal_estimate_expectation` takes a list of observables. Each is a list of Pauli strings (`I`, `X`, `Y`, `Z`, one letter per qubit) with one real coefficient per string. For example, `ZZI` is `{"paulis": ["ZZI"], "coefficients": [1.0]}`, and `0.5 ZZI + 0.5 IZZ` is `{"paulis": ["ZZI", "IZZ"], "coefficients": [0.5, 0.5]}`.

- The user may give a sparse form (letters, the qubits they act on, a coefficient). Expand it to full Pauli strings with `I` on the other qubits. Qubit 0 is the rightmost letter.
- To get one value per term, give each term as its own observable. To get one total, such as an energy, give one observable with all the terms.
- Check that each string's length matches the circuit's qubit count. For a non-symmetric observable, qubit ordering matters: if the user's intended ordering is unclear, confirm it against Fire Opal's [Pauli operators guide](https://docs.q-ctrl.com/fire-opal/execute/configure-circuits/pauli-operators-with-fire-opal) before submitting.

For variational loops that call this repeatedly, use `/fire-opal:iterate` instead.

Submitting uses hardware time. Never submit without the user's clear yes in the chat.

1. **Get OpenQASM text.** Read `.qasm` files. For a `.py` file, export its Qiskit circuits with the user's own Python, as `/fire-opal:validate` describes.
2. **Validate** (free): call `qctrl_fire_opal_validate`. Stop on problems.
3. **Submit.** State what will run (circuits, observables, backend, shots). Then ask the user to confirm and wait for a clear yes. Do not rely on the tool permission prompt: in some permission modes it does not appear. Then call `qctrl_fire_opal_estimate_expectation`. Pass `parameters` (one set per circuit) for parametric circuits, and `run_options` (`job_tags`, `session_id`, `reduce_approximation`) when the user asks.
4. **Wait.** Call `qctrl_get_job` until `SUCCESS`, `FAILURE` or `REVOKED`. Between checks, wait about 10 seconds, longer each time, with a background `sleep` command. Never call `qctrl_get_job` again straight away. If it's still queued after a few checks, give the job ID and point to `/fire-opal:result <job_id>`.
5. **Report.** Call `qctrl_fire_opal_get_circuit_results` per circuit and read `expectation_values` and `standard_deviations`. Report each value with its standard deviation (shot noise only), for example `⟨ZZI⟩ = 0.912 ± 0.009`. If the user has an exact or ideal value, compare against it.
