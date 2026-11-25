---
name: qaoa
description: Solve a binary optimization problem (Max-Cut, Max-k-Cut, a polynomial cost function, or a diagonal Ising Hamiltonian) on real hardware with Fire Opal's managed QAOA solver.
argument-hint: "<problem description, graph file, or cost function> [backend]"
disable-model-invocation: true
allowed-tools: mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_show_supported_devices mcp__plugin_fire-opal_qctrl__qctrl_list_credentials mcp__plugin_fire-opal_qctrl__qctrl_get_job mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_get_qaoa_results Bash(sleep:*)
---

# Solve an optimization problem with Fire Opal QAOA

Problem: `$ARGUMENTS`

Fire Opal's QAOA solver runs the whole hybrid loop (circuit construction, error suppression, execution, classical optimization) server-side on IBM Cloud or IonQ hardware. `qctrl_fire_opal_solve_qaoa` takes one `problem`, by `kind`:

- **`graph`**: nodes and edges (optionally weighted) for `problem_type` `maxcut` or `max-<k>-cut` (k ≥ 2).
- **`polynomial`**: a cost function over binary (0/1) variables to minimize, as a list of terms, each a coefficient and the variables it multiplies. For example, `3*x0*x1 - 2*x1*x2 + x0` has three terms. At least one term must multiply 2 or more variables: purely linear problems are rejected because they are classically trivial.
- **`pauli`**: a diagonal Ising Hamiltonian of `I`/`Z` Pauli strings with real coefficients, for example `ZZI + (-1) IZZ + 0.5 ZIZ`. Fire Opal converts it internally with Zᵢ = 1 − 2xᵢ.
- **Optional `constraints`**: one-hot groups, where exactly one variable in each group is 1. Name graph nodes by number, polynomial variables by name, and Ising qubits by index (0 is the rightmost letter). Ignored for k > 2 Max-k-Cut, which makes its own.

Steps:

1. **Formulate.** Turn the user's description into one of the forms above. If you have to translate a word problem into a cost polynomial (for example a QUBO with penalty terms), show the formulation and the variable meanings, and get a quick confirmation before submitting. Writing the problem to `fire-opal-results/problems/<name>.json` keeps it reusable.

2. **Pick the backend.** `backend_name` is required. For IBM Cloud, call `qctrl_fire_opal_show_supported_devices`. IonQ credentials cannot list devices, so ask the user for the IonQ backend name.

3. **Submit.** State the problem size (nodes and edges, or variables) and backend first. Then ask the user to confirm and wait for a clear yes. Do not rely on the tool permission prompt: in some permission modes it does not appear. Then call `qctrl_fire_opal_solve_qaoa`. On IBM, pass `run_options` (`job_tags`, `session_id`, `reduce_approximation`) when the user asks.

4. **Wait.** QAOA runs many iterations and usually takes longer than a single execution. Call `qctrl_get_job` until `SUCCESS`, `FAILURE` or `REVOKED`. Between checks, wait about 30 seconds, longer each time, with a background `sleep` command. Never call `qctrl_get_job` again straight away. If it's still running after a few checks, give the job ID and point to `/fire-opal:result <job_id>`.

5. **Interpret.** Call `qctrl_fire_opal_get_qaoa_results`. Report the solution bitstring and its cost, translated back into the user's terms using `variables_to_bitstring_index_map` (for Max-Cut: which nodes fall on each side and how many edges are cut). Include the iteration count and the top of the final distribution. For small problems (≲ 20 variables), offer to brute-force the classical optimum to compare.
