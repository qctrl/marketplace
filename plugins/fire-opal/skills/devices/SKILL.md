---
name: devices
description: List the quantum backends Fire Opal supports for the user's saved IBM Cloud or Amazon Braket credentials. Use when the user asks which Fire Opal devices or backends they can run on.
argument-hint: "[credential name]"
allowed-tools: mcp__plugin_fire-opal_qctrl__qctrl_list_credentials mcp__plugin_fire-opal_qctrl__qctrl_fire_opal_show_supported_devices
---

# List Fire Opal devices

Credential requested: `$ARGUMENTS` (blank means the user's only credential, or ask).

1. If the user has more than one credential and did not name one, call `qctrl_list_credentials` and ask which to use. IonQ credentials cannot list devices: through this server, IonQ runs only the QAOA solver.
2. Call `qctrl_fire_opal_show_supported_devices`, with `credential_name` when you have one.
3. List the devices as a compact list, with the credential and provider they are for.

If the call says the user has no credential, point to `/fire-opal:setup`. If it says the user is not signed in, tell them to run `/mcp` and sign in to `plugin:fire-opal:qctrl`.
