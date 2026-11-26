---
name: fire-opal-reference
description: Reference for writing Python code with Q-CTRL's fireopal package (authentication, credentials, validate, execute, estimate_expectation, iterate, iterate_expectation, stop_iterate, solve_qaoa, simulate_dynamics, integrate_monte_carlo and its distribution/objective helpers, PauliOperator, run options, job retrieval) and Fire Opal's circuit requirements. Use when writing, reviewing or debugging code that imports fireopal, or answering questions about the Fire Opal API.
user-invocable: false
---

# Fire Opal Python API reference

Package: `pip install fire-opal` → `import fireopal as fo`. Requires an active Fire Opal subscription. Docs: <https://docs.q-ctrl.com/fire-opal> · API: <https://docs.q-ctrl.com/references/fire-opal/fireopal>

To run jobs from Claude Code, prefer this plugin's commands. They run Fire Opal through the Q-CTRL MCP server, so the user needs no local package or API key:

| Command | Wraps |
| --- | --- |
| `/fire-opal:validate` | `validate` |
| `/fire-opal:run` | `execute` |
| `/fire-opal:expectation` | `estimate_expectation` |
| `/fire-opal:iterate` | `iterate`, `iterate_expectation`, `stop_iterate` |
| `/fire-opal:qaoa` | `solve_qaoa` |
| `/fire-opal:dynamics` | `simulate_dynamics` |
| `/fire-opal:monte-carlo` | `integrate_monte_carlo` |
| `/fire-opal:result`, `/fire-opal:jobs` | `get_result`, job status and history |

Use this reference when the user is writing their own code. For the current docs, the `qctrl_search_docs` and `qctrl_get_doc` tools search and read docs.q-ctrl.com.

For full signatures, parameter meanings, return formats and worked examples of every function, read [references/api.md](references/api.md).

## Essentials

```python
import fireopal as fo
fo.authenticate_qctrl_account(api_key="YOUR_API_KEY")     # locally, a browser login also works
creds = fo.credentials.make_credentials_for_ibm_cloud(token="YOUR_IBM_CLOUD_API_KEY",
                                                     instance="YOUR_IBM_CLOUD_CRN")
backends = fo.show_supported_devices(credentials=creds)["supported_devices"]
```

The values above are placeholders. Never hard-code real API keys in code you write; let the user supply them in their own environment. User code runs in the user's own Python environment, with `fire-opal` installed there; this plugin installs nothing and keeps no keys locally.

**Job objects.** Every job function except `validate` returns a `FireOpalJob` immediately. It has:

- `.action_id`
- `.status()`, which returns a dict with `action_status`: SUCCESS, FAILURE, REVOKED, PENDING, RECEIVED, RETRY or STARTED
- `.result()`, which blocks until the job finishes

To fetch a job later, use `fo.get_result(action_id)`. To list jobs, use `fo.activity_monitor(limit=, offset=, status=)` or `fo.get_action_metadata(...)`. Always take results from Fire Opal rather than the provider's console: provider-side results lack Fire Opal's post-processing.

**Circuits** are OpenQASM 2.0/3.0 strings. Convert with `qiskit.qasm2.dumps(qc)`, or with `qiskit.qasm3.dumps(qc)` for circuits with unbound `Parameter`s. Submit logical (un-transpiled) circuits: Fire Opal handles layout, transpilation, error suppression and readout post-processing.

**Requirements** (checked by `fo.validate`, which is free):

- at least one measurement and one quantum register
- no physical-qubit specifications or ancillas, and no conditional operations
- fewer than 20,000 multi-qubit gates
- mid-circuit measurements and multiple classical registers only on IBM
- no more qubits than the backend has
- at most 300 circuits per job; use `iterate` for more

### Choosing a function

| Goal | Function |
| --- | --- |
| Bitstring probabilities, one-off | `execute` (returns probabilities) |
| Expectation values, one-off | `estimate_expectation` |
| Repeated submissions or > 300 circuits | `iterate` (returns **counts**) or `iterate_expectation`, then **always** `stop_iterate(credentials, backend_name)` in a `finally:` |
| Combinatorial optimization | `solve_qaoa`, which takes an `nx.Graph` with `problem_type`, a `sympy.Expr`, or a diagonal `PauliOperator` |
| Fermi-Hubbard time evolution | `simulate_dynamics` (1D, Trotter) |
| Expectation of f(X) under a distribution (pricing, risk) | `integrate_monte_carlo` with `make_monte_carlo_problem(objective, distribution)` |

**Provider options.** The only one today is `fireopal.run_options.IbmRunOptions(session_id=None, job_tags=None, reduce_approximation=False)`, passed as `run_options=` to `validate`, `execute`, `estimate_expectation`, `iterate*`, `solve_qaoa` and `integrate_monte_carlo`. `simulate_dynamics` doesn't take run options.
