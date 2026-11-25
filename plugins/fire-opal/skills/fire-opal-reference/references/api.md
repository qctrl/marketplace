# fireopal API — detailed reference

Signatures below were taken from the installed `fire-opal` 12.4.0 package. If the user's version differs, check with `inspect.signature(fo.<name>)`, or run `print_package_versions()`.

## Contents

1. Authentication and credentials
2. Devices and validation
3. execute / estimate_expectation
4. iterate / iterate_expectation / stop_iterate
5. solve_qaoa
6. simulate_dynamics
7. integrate_monte_carlo and helpers
8. Types: PauliOperator, ModelType, SimulationType, SparseOccupationOperators
9. Run options
10. Jobs and results

---

## 1. Authentication and credentials

```python
fo.authenticate_qctrl_account(api_key: str | None = None) -> None
fo.configure_organization(organization_slug: str) -> None          # only if the account has several orgs

fo.credentials.make_credentials_for_ibm_cloud(token: str, instance: str)   # IBM Cloud API key + instance CRN
fo.credentials.make_credentials_for_ionq(token: str)
fo.credentials.make_credentials_for_braket(arn: str)                # IAM role with Braket permissions that trusts Q-CTRL's AWS account
```

The credential helpers contact Q-CTRL to attach your organization ID, so authenticate first.

## 2. Devices and validation

```python
fo.show_supported_devices(credentials) -> {"supported_devices": [...], "warnings": [...]}

fo.validate(circuits: list[str], credentials, backend_name: str, run_options=None) -> dict
# -> {"results": [...per-circuit validation details/errors...], "warnings": [...]}
# Free. In current versions it returns the dict directly; older docs show a FireOpalJob.
```

## 3. execute / estimate_expectation

```python
fo.execute(circuits: str | list[str], shot_count: int, credentials, backend_name: str,
           parameters: dict[str, float] | list[dict[str, float]] | None = None, run_options=None) -> FireOpalJob
# result(): {"results": [{bitstring: probability}, ...one per circuit],
#            "provider_job_ids": [...], "execution_metadata": {...}, "warnings": [...]}

fo.estimate_expectation(circuits, shot_count, credentials, backend_name, observables,
                        parameters=None, run_options=None) -> FireOpalJob
# observables: "ZZI" | ["ZZI", "IZZ"] | {"ZZI": 0.5, "IZZ": 0.5} | [("ZZI", 0.5)] | PauliOperator | [PauliOperator]
# result(): {"expectation_values": [...], "standard_deviations": [...]}   # std devs = shot noise only
```

`parameters`: a single dict binds the same values in every circuit; a list gives one dict per circuit.

## 4. iterate / iterate_expectation / stop_iterate

Same arguments as `execute` / `estimate_expectation`. Fire Opal keeps a provider session open (on IBM, a Qiskit Runtime session) so repeated calls skip queueing and preprocessing. Use these for variational loops and for batches of more than 300 circuits.

```python
fo.iterate(circuits, shot_count, credentials, backend_name, parameters=None, run_options=None) -> FireOpalJob
# result(): {"results": [{bitstring: COUNT}, ...]}         # counts, not probabilities

fo.iterate_expectation(circuits, shot_count, credentials, backend_name, observables,
                       parameters=None, run_options=None) -> FireOpalJob
# result(): {"expectation_values": [...], "standard_deviations": [...]}

fo.stop_iterate(credentials, backend_name: str) -> None    # ALWAYS call when finished
```

VQE pattern:

```python
from scipy.optimize import minimize
qasm = qiskit.qasm3.dumps(ansatz)                     # ansatz with Parameter("theta0"), ...
names = sorted(p.name for p in ansatz.parameters)
H = {"ZZ": 1.0, "XX": 0.5, "YY": 0.5}

def energy(x):
    job = fo.iterate_expectation(circuits=[qasm], shot_count=4096, credentials=creds, backend_name=backend,
                                 observables=H, parameters=dict(zip(names, map(float, x))))
    return job.result()["expectation_values"][0]

try:
    res = minimize(energy, x0=[0.1] * len(names), method="COBYLA", options={"maxiter": 40})
finally:
    fo.stop_iterate(credentials=creds, backend_name=backend)
```

## 5. solve_qaoa

```python
fo.solve_qaoa(problem, credentials, problem_type: str | None = None, backend_name: str | None = None,
              run_options=None, constraint: dict[tuple, int] | None = None) -> FireOpalJob
```

`problem` takes one of three forms:

- `nx.Graph` with `problem_type="maxcut"` or `"max-<k>-cut"` (k ≥ 2)
- `sympy.Expr`, a polynomial over binary variables to **minimize**; it must contain at least one nonlinear term
- `PauliOperator` with only I/Z terms and real coefficients (Ising); converted internally with Zᵢ = 1 − 2xᵢ

`constraint` gives one-hot groups, such as `{(0, 1): 1, (4, 5, 6): 1}` or `{("x", "y", "z"): 1}`. It is ignored for Max-k-Cut with k > 2.

QAOA runs on IBM or IonQ.

`result()` keys: `solution_bitstring`, `solution_bitstring_cost`, `final_bitstring_distribution` (counts), `iteration_count`, `variables_to_bitstring_index_map`, `best_parameters`, `warnings`.

Method: Sachdeva et al. 2024, arXiv:2406.01743.

## 6. simulate_dynamics

```python
from fireopal.types import ModelType, SimulationType, SparseOccupationOperators

fo.simulate_dynamics(model: ModelType, initial_state: str, simulation: SimulationType,
                     observables: SparseOccupationOperators | list[SparseOccupationOperators],
                     shot_count: int, backend_name: str, credentials) -> FireOpalJob
```

- `initial_state`: one character per site (e empty, u up, d down, b both). The **rightmost character is site 0**, and there are at most 64 sites.
- `ModelType(hopping_amplitude, on_site_interaction, chemical_potential, name="Fermi-Hubbard", lattice_type="1D")`. Each field is a float or a list of floats.
- `SimulationType(time_step: float, step_count: int, method="trotter")`
- `SparseOccupationOperators(operators=[[("Nup", 3)], [("Ndn", 7), ("Nup", 9)]], coefficients=[1.0, -1.0])`. Each term is a product of number operators. Spins are "Nup" and "Ndn", and sites must be less than `len(initial_state)`.

`result()` keys:

- `observable_expectation_values`: a list per observable, with values at t = 0, dt, …, step_count·dt
- `observables`
- `provider_job_ids`
- `warnings`

Example (docs): `"eeeu"`, J = 1, U = 0, μ = 0, dt = 0.2, 8 steps, with one `[("Nup", site)]` observable per site. This returns a 4 × 9 array of spin-up occupations.

## 7. integrate_monte_carlo and helpers

```python
fo.integrate_monte_carlo(problem: dict, credentials, backend_name: str | None = None,
                         integrator_options: dict | None = None, run_options=None) -> FireOpalJob
# problem: fo.make_monte_carlo_problem(objective_function_parameters=..., distribution_parameters=...)
#      or  {"preparation_circuit": <QASM>}   (probability encoded on a 1-qubit register named target/output; backend required)
# integrator_options: {"max_iteration_count": int, "target_variance": float, "full_output": bool}
# result(): {"estimate": float, "variance": float, "iteration_count": int, "metadata": {...}}   # metadata["iterations"] if full_output

fo.make_monte_carlo_problem(objective_function_parameters: dict, distribution_parameters: dict) -> dict
```

**Distributions.** `uncertainty` is the number of points per dimension, at least 2. `sigma` is a **variance** (or a covariance matrix):

```python
fo.make_uniform_distribution(uncertainty: int)
fo.make_normal_distribution(uncertainty, mu=None, sigma=None, bounds=None, upto_diag=False)       # defaults 0, 1, (-1, 1)
fo.make_lognormal_distribution(uncertainty, mu=None, sigma=None, bounds=None, upto_diag=False)    # mu/sigma of log X; bounds default (0, 1)
fo.make_gci_model_distribution(latent_variable, normal_max_value, p_zeros: list[float], rhos: list[float])  # credit risk
```

For multivariate distributions, use lists: `uncertainty=[4, 4]`, `mu=[0, 0]`, `sigma=[[1, 0], [0, 1]]`, `bounds=[(-1, 1), (-1, 1)]`. `upto_diag=True` gives a shallower circuit, but is valid only if the objective ignores relative phases.

**Objectives:**

```python
fo.make_european_call_pricing_objective(strike_price, rescaling_factor, bounds=None)
fo.make_european_call_delta_objective(strike_price, bounds=None)
fo.make_fixed_income_pricing_objective(pca_matrix: np.ndarray (T×D), initial_interests: list (T), cash_flow: list (T),
                                       rescaling_factor, bounds: list[tuple] (D) | None = None)
```

Docs example: `make_uniform_distribution(8)` with `make_european_call_delta_objective(0.5, bounds=(0, 1))` and `max_iteration_count=10`. The exact answer is 0.5; the hardware gave 0.4874.

## 8. Types

```python
from fireopal.types import PauliOperator
PauliOperator("ZZI")                                    # coefficient 1
PauliOperator(["ZZI", "IZZ"], coefficients=[0.5, 0.5])
PauliOperator.from_dict({"ZZI": 0.5, "IZZ": 0.5})
PauliOperator.from_list([("ZZI", 0.5), ("IZZ", 0.5)], qubit_count=None)
PauliOperator.from_sparse_list([("ZZ", [0, 7], 1.0), ("X", [3], -0.5)], qubit_count=12)   # local terms on given qubits
op.to_dict(); op.to_list(); op.coefficients
```

For non-symmetric strings, check the qubit ordering against the docs' [Pauli operators guide](https://docs.q-ctrl.com/fire-opal/execute/configure-circuits/pauli-operators-with-fire-opal).

`ModelType`, `SimulationType` and `SparseOccupationOperators` are described in section 6. Each has `to_dict()` and `from_dict()`.

## 9. Run options

```python
from fireopal.run_options import IbmRunOptions
IbmRunOptions(session_id: str | None = None, job_tags: list[str] | None = None, reduce_approximation: bool = False)
```

- `session_id`: run within an existing IBM Runtime session
- `job_tags`: tags shown in the IBM Quantum dashboard
- `reduce_approximation`: keep near-identity gates, for circuits where tiny rotations matter

These are IBM only. Pass them with `run_options=` to `validate`, `execute`, `estimate_expectation`, `iterate`, `iterate_expectation`, `solve_qaoa` and `integrate_monte_carlo`.

## 10. Jobs and results

```python
job = fo.FireOpalJob(action_id)      # re-attach to any job
job.status()   # {"action_status": "SUCCESS", "status_message": "..."}
job.result()   # blocks until done
fo.get_result(action_id)             # same dict as job.result()
fo.activity_monitor(limit=1, offset=0, status=None)        # prints a table
fo.get_action_metadata(limit=1, offset=0, status=None)     # list of ActionMetadata(name, status, created_at, updated_at, model_id)
fo.print_package_versions()
```

Job results stay retrievable after the Python session ends.
