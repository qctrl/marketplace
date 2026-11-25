---
name: validate
description: Check quantum circuits (OpenQASM files or Qiskit circuits in a Python file) against Fire Opal's requirements for a backend, without using hardware time. Use before running circuits with Fire Opal, or when the user asks whether a circuit is Fire Opal compatible.
argument-hint: "<circuit.qasm|circuits_dir|circuit.py> [backend]"
allowed-tools: mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_validate mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_show_supported_devices mcp__plugin_fire-opal_qctrl__qctrl_list_credentials
---

# Validate circuits with Fire Opal

Arguments: `$ARGUMENTS`. The first argument is the circuit source; an optional second argument is the backend name. If no circuit source was given, look for `.qasm` files or a Python file defining Qiskit circuits in the working directory and ask which to use if it is ambiguous.

1. **Get OpenQASM text.** The MCP tools take each circuit as OpenQASM 2.0 or 3.0 text, under 1 MB each, at most 300 per call.

   - `.qasm` files or a directory of them: read them.
   - A `.py` file: export its module-level `QuantumCircuit` objects with the user's own Python, which must have Qiskit installed:

     ```bash
     python - path/to/file.py <<'EOF'
     import pathlib, runpy, sys
     from qiskit import QuantumCircuit, qasm2, qasm3
     out = pathlib.Path("fire-opal-results/circuits")
     out.mkdir(parents=True, exist_ok=True)
     for name, qc in runpy.run_path(sys.argv[1]).items():
         if isinstance(qc, QuantumCircuit):
             text = qasm3.dumps(qc) if qc.parameters else qasm2.dumps(qc)
             (out / f"{name}.qasm").write_text(text)
             measured = "measure" in qc.count_ops()
             print(name, qc.num_qubits, sorted(p.name for p in qc.parameters), "measured" if measured else "NO MEASUREMENTS")
     EOF
     ```

     Running the file runs the user's code, so say which file you run. Warn about any circuit with no measurements: Fire Opal requires at least one.

2. **Pick the backend.** Use the one the user named. Otherwise call `qctrl_fire_opal_show_supported_devices` and ask.

3. **Validate** (free, no hardware time): call `qctrl_fire_opal_validate` with the circuits and `backend_name`. For parametric circuits, pass the same `parameters` you will run with.

4. **Report.** Say whether the circuits are valid. Quote each problem and warning. Fire Opal stops at the first invalid circuit, so fix it and validate again to find the next one.

5. **If validation fails**, explain the likely fix using Fire Opal's circuit requirements:

   - valid QASM 2.0/3.0 with at least one measurement and one quantum register
   - no physical-qubit specifications or ancilla qubits, no conditional operations
   - fewer than 20,000 multi-qubit gates
   - mid-circuit measurements and multiple classical registers only on IBM backends
   - no more qubits than the backend has

   Offer to fix the circuit. Don't edit the user's source files unless they agree.

If a call fails because the user is not signed in, tell them to run `/mcp` and sign in to `plugin:fire-opal:qctrl`. If they have no credential, point to `/fire-opal:setup`.
