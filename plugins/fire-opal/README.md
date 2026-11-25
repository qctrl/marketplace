# Fire Opal plugin for Claude Code

Run quantum circuits on real hardware with [Q-CTRL Fire Opal](https://q-ctrl.com/fire-opal)'s automated error suppression, directly from Claude Code.

The plugin runs Fire Opal through the Q-CTRL MCP server. You sign in with your Q-CTRL account, and your hardware credentials are saved on the server. Nothing is installed on your computer, and no API keys are kept there.

## Commands

| Command | What it does | Uses hardware time? |
| --- | --- | --- |
| `/fire-opal:setup` | Signs you in, checks your organization's Fire Opal access, and saves hardware credentials | No |
| `/fire-opal:devices [credential]` | Lists the backends Fire Opal supports for your credentials | No |
| `/fire-opal:validate <circuits> [backend]` | Checks circuits against Fire Opal's requirements | No (free) |
| `/fire-opal:run <circuits> [backend] [shots]` | Validates, submits, waits, and summarizes bitstring probabilities | Yes |
| `/fire-opal:expectation <circuits> <observables> [backend] [shots]` | Estimates expectation values ± standard deviation | Yes |
| `/fire-opal:iterate <sweep\|vqe\|stop> …` | Parameter sweeps, batches of more than 300 circuits, and variational loops (VQE etc.) in one reusable provider session, always released afterward | Yes |
| `/fire-opal:qaoa <problem> [backend]` | Solves Max-Cut / Max-k-Cut / polynomial / Ising problems with the managed QAOA solver | Yes |
| `/fire-opal:dynamics <initial state> …` | Simulates 1D Fermi-Hubbard time evolution and tracks site occupations or custom observables | Yes |
| `/fire-opal:monte-carlo <problem>` | Quantum Monte Carlo integration: option price or delta, fixed income, credit risk, or a custom preparation circuit | Yes |
| `/fire-opal:result <job_id>` | Fetches and summarizes a job's result, and saves the full result on request | No |
| `/fire-opal:jobs [limit] [status]` | Lists recent jobs | No |

`<circuits>` can be `.qasm` files, a directory of them, or a Python file that defines Qiskit `QuantumCircuit` objects at module level. Claude exports a Python file to QASM with your own Python, so Qiskit must be installed there.

On IBM backends, the submitting commands except `dynamics` also accept run options: job tags for the IBM dashboard, an existing Runtime session ID, or reduced-approximation compilation. Ask for them in plain language, e.g. "tag these jobs `calibration-oct`".

The plugin also includes a background reference skill covering the whole `fireopal` API: every function, type and run option. Claude uses it when you ask it to write or debug your own Fire Opal code, which runs in your own Python environment.

Commands that submit jobs (`run`, `expectation`, `iterate`, `qaoa`, `dynamics`, `monte-carlo`) run only when you type them, and Claude Code asks you to approve the submit tool call before anything is sent to hardware. Validation, status checks and result retrieval run without prompts.

### Providers

| Provider      | Commands                                             |
| ------------- | ---------------------------------------------------- |
| IBM Cloud     | All                                                  |
| Amazon Braket | `devices`, `validate`, `run` (no circuit parameters) |
| IonQ          | `qaoa`                                               |

### Fire Opal function coverage

| Fire Opal function | Command | MCP tool |
| --- | --- | --- |
| `show_supported_devices` | `/fire-opal:devices` | `qctrl_fire_opal_show_supported_devices` |
| `validate` | `/fire-opal:validate` | `qctrl_fire_opal_validate` |
| `execute` | `/fire-opal:run` | `qctrl_fire_opal_execute` |
| `estimate_expectation` | `/fire-opal:expectation` | `qctrl_fire_opal_estimate_expectation` |
| `iterate`, `iterate_expectation`, `stop_iterate` | `/fire-opal:iterate` | `qctrl_fire_opal_iterate`, `qctrl_fire_opal_iterate_expectation`, `qctrl_fire_opal_stop_iterate` |
| `solve_qaoa` | `/fire-opal:qaoa` | `qctrl_fire_opal_solve_qaoa` |
| `simulate_dynamics` | `/fire-opal:dynamics` | `qctrl_fire_opal_simulate_dynamics` |
| `integrate_monte_carlo` and its distribution and objective helpers | `/fire-opal:monte-carlo` | `qctrl_fire_opal_integrate_monte_carlo` |
| `get_result`, `FireOpalJob.status` | `/fire-opal:result` | `qctrl_get_job`, `qctrl_fire_opal_get_*_results`, `qctrl_get_results_link` |
| `activity_monitor` | `/fire-opal:jobs` | `qctrl_list_jobs` |
| `authenticate_qctrl_account`, `configure_organization`, credentials helpers | `/fire-opal:setup` | `qctrl_whoami`, `qctrl_add_credential`, `qctrl_list_credentials`, `qctrl_remove_credentials` |
| `run_options.IbmRunOptions` | Submitting commands except `dynamics` | `run_options` |

Claude saves files to `fire-opal-results/` in your project when you ask for them: exported circuits, sweep results, and full job results (`<job_id>.json`).

## Install

Add the Q-CTRL marketplace and install the plugin:

```text
/plugin marketplace add qctrl/marketplace
/plugin install fire-opal@qctrl
```

To develop the plugin, load it directly from the root of the [marketplace](https://github.com/qctrl/marketplace) repository:

```bash
claude --plugin-dir plugins/fire-opal
```

Then, in the Claude Code session, sign in and set up:

```text
/mcp
/fire-opal:setup
/fire-opal:devices
```

In `/mcp`, select `plugin:fire-opal:qctrl` and sign in with your Q-CTRL account in the browser.

## How it works

- `.mcp.json` adds the Q-CTRL MCP server (`qctrl`) with the public OAuth client `mcp-login`. Claude Code signs you in with your Q-CTRL account and keeps the token. It points at the dev server (`https://mcp-server.private.dev.q-ctrl.com/mcp`) until the production server (`https://mcp.q-ctrl.com/mcp`) is live.
- Each skill tells Claude which MCP tools to call and how to explain the results. Read-only tools are pre-approved in each skill's `allowed-tools`; tools that submit jobs are not.
- Hardware credentials are saved on the server through a secure Q-CTRL page (`qctrl_add_credential`), never in the chat.
- Tools that submit a job return a job ID straight away. Claude checks `qctrl_get_job` until it finishes, then calls the results tool it names.

## Limits

- Each job runs at most 300 circuits. `/fire-opal:iterate` splits larger sweeps into several steps in one session.
- A variational loop is one tool call per step, so long loops are slow. `/fire-opal:iterate` evaluates many parameter points in each step to keep the number of steps small, and agrees a step limit (at most 500) with you.

## Notes

- **No local setup.** `/fire-opal:setup` installs no Python packages, and there is no `/plugin configure`. You sign in with your Q-CTRL account, and hardware credentials are saved on the server.
- **IonQ runs QAOA only.** The MCP server supports IonQ for `qaoa`. Use IBM Cloud or Amazon Braket for the other commands.
- **No default provider, backend or shot count.** Claude asks for the backend, or lists your devices, and suggests 2048 shots.
- **Results are saved only on request.** There is no automatic `fire-opal-results/<id>.json` for each job and no `jobs.jsonl` log. Ask Claude to save a result, or use `/fire-opal:jobs` to see past jobs.
- **`/fire-opal:result` takes no wait time.** It checks the job once. Ask Claude to keep checking if you want it to wait.
- **Variational loops have no local script.** Claude is the optimizer and calls one tool per step, with at most 500 steps. To run your own optimizer in Python, install the `fire-opal` Python package in your environment and use the `fireopal` package directly.

## Requirements

- Claude Code with plugin and MCP OAuth support
- A Q-CTRL account in an organization with Fire Opal, and credentials for at least one supported hardware provider
- For Python circuit files: Python with Qiskit
