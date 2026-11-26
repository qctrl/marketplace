---
name: dynamics
description: Simulate time evolution of a 1D Fermi-Hubbard chain on quantum hardware with Fire Opal's managed dynamics simulator (simulate_dynamics), and measure site occupations or custom occupation observables over time.
argument-hint: "<initial state, e.g. eeeu> [hopping U mu dt steps] [backend]"
disable-model-invocation: true
---

# Quantum dynamics simulation with Fire Opal

Request: `$ARGUMENTS`

Fire Opal's dynamics simulator builds, error-suppresses and runs the Trotterized circuits itself. The user describes physics, not circuits. Currently supported: the **1D Fermi-Hubbard model** with **Trotter** time evolution, on IBM Cloud credentials.

## Inputs to settle

| Input | Meaning | `qctrl_fire_opal_simulate_dynamics` field |
| --- | --- | --- |
| Initial state | One character per site: `e` empty, `u` spin up, `d` spin down, `b` both. **The rightmost character is site 0.** Up to 64 sites. | `initial_state` |
| Hopping amplitude _J_ | One number for every bond, or one per bond (one fewer than the sites) | `model.hopping_amplitude` |
| On-site interaction _U_ | One number for every site, or one per site from site 0 | `model.on_site_interaction` |
| Chemical potential _μ_ | One number for every site, or one per site from site 0 | `model.chemical_potential` |
| Time step _dt_ and step count | Results are returned at _t_ = 0, dt, …, steps × dt. At most 100 steps. | `simulation.time_step`, `simulation.step_count` |
| Observables | Sums of products of number operators, 1 to 100 | `observables` |

### Observables

Each observable is a list of `operators` with one coefficient each. Each operator is a product of occupations, each a `spin` (`Nup` or `Ndn`) and a `site` (0 to length − 1). For example:

- Per-site total occupation ⟨n↑ᵢ + n↓ᵢ⟩: one observable per site _i_, with operators `[[{"spin": "Nup", "site": i}], [{"spin": "Ndn", "site": i}]]`, coefficients `[1, 1]`.
- Double occupancy on site 1: operators `[[{"spin": "Nup", "site": 1}, {"spin": "Ndn", "site": 1}]]`, coefficients `[1]`.
- n↑(0) − n↑(3): operators `[[{"spin": "Nup", "site": 0}], [{"spin": "Nup", "site": 3}]]`, coefficients `[1, -1]`.

Keep your own label for each observable: the results come back in the same order.

If the user gives physics in words ("one up-spin at the left end of a 6-site chain, U = 2"), translate it and show the mapping, especially which end is site 0, before submitting. Ask about anything that changes the physics and isn't stated, such as U or the total evolution time. Defaults to suggest if they have none: J = 1, U = 0, μ = 0, dt = 0.2, 8 steps, total occupation per site.

Keep circuits shallow: the number of Trotter steps drives circuit depth, so long evolutions on many sites lose fidelity. For a first run, suggest about ≤ 10 steps.

## Run

1. Pick the backend with `qctrl_fire_opal_show_supported_devices`, unless the user named one. Suggest 2048 shots.
2. State the configuration in one compact block: lattice, initial state, J/U/μ, dt × steps = total time, observables, shots, backend. Then ask the user to confirm and wait for a clear yes. Do not rely on the tool permission prompt: in some permission modes it does not appear. Then call `qctrl_fire_opal_simulate_dynamics`.
3. Wait: call `qctrl_get_job` until `SUCCESS`, `FAILURE` or `REVOKED`. Between checks, wait about 30 seconds, longer each time, with a background `sleep` command. Never call `qctrl_get_job` again straight away. If the job is still queued after a few checks, give the job ID and point to `/fire-opal:result <job_id>`.

## Report

Call `qctrl_fire_opal_get_dynamics_results`. It returns one time series per observable, in the order you sent them, from _t_ = 0.

- Present a compact table (observables × selected times) and describe the physics. For example: how occupation spreads from the initial site, and whether total particle number is conserved. The sum of total occupations should stay at the initial particle count, so a drift is a quick hardware-noise check.
- Offer a plot of occupation vs. time per site, or a site × time heatmap, saved under `fire-opal-results/`.
- For small chains (≲ 8 sites), offer to compute the exact classical evolution with numpy/scipy to compare.
