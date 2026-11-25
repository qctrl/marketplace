---
name: monte-carlo
description: Estimate expectation values with quantum Monte Carlo integration on hardware via Fire Opal's integrate_monte_carlo, e.g. European call option price or delta, fixed-income pricing, or credit risk with a Gaussian conditional independence model, or a custom state-preparation circuit.
argument-hint: "<problem description or preparation circuit> [backend]"
disable-model-invocation: true
allowed-tools: mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_show_supported_devices mcp__plugin_fire-opal_qctrl__qctrl_list_credentials mcp__plugin_fire-opal_qctrl__qctrl_get_job mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_get_monte_carlo_results Bash(sleep:*)
---

# Monte Carlo integration with Fire Opal

Request: `$ARGUMENTS`

`qctrl_fire_opal_integrate_monte_carlo` estimates E[f(X)] for a discretized probability distribution of X and an objective f. Fire Opal builds the circuits, runs the iterative integrator with error suppression, and returns an estimate and its variance. It runs on IBM Cloud credentials. A `problem` is either `kind` `built_in` (a distribution plus an objective) or `kind` `circuit` (a custom preparation circuit).

## Option 1: built-in distribution + objective

Each takes a `name` and that helper's parameters.

**Distributions.** `uncertainty` is the number of discrete points per dimension (2 to 4096; 8 points = 3 qubits per dimension):

| name | Parameters |
| --- | --- |
| `uniform` | `uncertainty` |
| `normal` | `uncertainty`, `mu` (default 0), `sigma` (**variance**, default 1; covariance matrix if multivariate), `bounds` (default [-1, 1]), `upto_diag` |
| `lognormal` | as `normal`, but `mu`/`sigma` are the mean/**variance of log X**; `bounds` default [0, 1] |
| `gaussian_conditional_independence_model` | `latent_variable` (points for the latent normal), `normal_max_value`, `p_zeros` (default probability per asset), `rhos` (sensitivity per asset, same length) |

Use lists for multivariate cases: `uncertainty: [4, 4]`, `mu: [0, 0]`, `sigma: [[1, 0], [0, 1]]`, `bounds: [[-1, 1], [-1, 1]]`.

`upto_diag: true` gives a shallower circuit, but only when the objective doesn't depend on the relative phases. Only suggest it when the user is chasing fidelity and understands the caveat.

**Objectives:**

| name | Parameters |
| --- | --- |
| `european_call_pricing` | `strike_price`, `rescaling_factor` (approximation scaling of the linear payoff), `bounds` |
| `european_call_delta` | `strike_price`, `bounds` |
| `fixed_income_pricing` | `pca_matrix` (T × D), `initial_interests` (length T), `cash_flow` (length T), `rescaling_factor`, `bounds` (length D) |

The objective's `bounds` should match the distribution's `bounds`, so that both refer to the same discretized grid.

## Option 2: custom preparation circuit

Pass `preparation_circuit` as OpenQASM 2.0 text (for Qiskit, `qiskit.qasm2.dumps(qc)` in the user's own Python). The circuit must encode the target probability in a single-qubit register named `target` or `output`.

## Integrator options

- `max_iteration_count` (1 to 100): a good starter is 10; Fire Opal recommends at most 20. More iterations give more precision and use more hardware time.
- `integrator_options.target_variance`: stop once the variance falls below this.
- `integrator_options.full_output`: include per-iteration estimates.

## Run

1. Translate the request into a distribution and objective, or a circuit. Show the user the discretization: the number of points, the bounds, and what the estimated quantity means in their units. Watch for **variance vs. standard deviation** in `sigma`, which is easy to get wrong. Ask about anything that changes the answer and isn't stated, such as the strike, the volatility, or the bounds.
2. Pick the backend with `qctrl_fire_opal_show_supported_devices`, unless the user named one.
3. State the plan in one block: the problem, qubits per dimension, max iterations, and backend. Then ask the user to confirm and wait for a clear yes. Do not rely on the tool permission prompt: in some permission modes it does not appear. Then call `qctrl_fire_opal_integrate_monte_carlo`. Pass `run_options` (`job_tags`, `session_id`, `reduce_approximation`) when the user asks.
4. Wait: call `qctrl_get_job` until `SUCCESS`, `FAILURE` or `REVOKED`. Between checks, wait about 30 seconds, longer each time, with a background `sleep` command. Never call `qctrl_get_job` again straight away. If the job is still queued after a few checks, give the job ID and point to `/fire-opal:result <job_id>`.

## Report

Call `qctrl_fire_opal_get_monte_carlo_results`.

- Give the `estimate ± standard_deviation` and the iteration count.
- Say what the estimate is: Fire Opal returns E[f(X)] over the discretized, bounded grid. For `european_call_pricing`, the objective uses a linear approximation scaled by `rescaling_factor`. Don't present the raw estimate as a dollar price unless the problem was set up so that it is one. Explain any conversion step you apply, and flag that you applied it.
- Offer a classical check, which is cheap for these sizes: compute E[f(X)] exactly on the same discretized grid with numpy, and report the absolute difference. For example, the docs' uniform-8 / delta-at-0.5 problem has an exact answer of 0.5.
