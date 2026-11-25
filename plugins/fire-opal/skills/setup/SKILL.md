---
name: setup
description: Sign in to the Q-CTRL MCP server, check the user's Q-CTRL organization and Fire Opal access, and add, replace or remove hardware-provider credentials on the server. Use when the user wants to set up Fire Opal, sign in to Q-CTRL, or add hardware credentials.
allowed-tools: mcp__plugin_fire-opal_qctrl__qctrl_whoami mcp__plugin_fire-opal_qctrl__qctrl_list_credentials mcp__plugin_fire-opal_qctrl__qctrl_add_credential
---

# Fire Opal setup

Get the user from zero to a working Fire Opal connection. Arguments, if any: `$ARGUMENTS`.

The plugin runs Fire Opal through the Q-CTRL MCP server (`qctrl`). Nothing is installed locally, and no API keys are kept on the user's computer.

1. **Sign in.** Call `qctrl_whoami`.

   - If the `qctrl` tools are not available, or the call fails because the user is not signed in, tell the user to run `/mcp`, select `plugin:fire-opal:qctrl` and sign in with their Q-CTRL account in the browser. Then call `qctrl_whoami` again.
   - Report the user's name, the organization they work in, and whether that organization can use Fire Opal. If they belong to several organizations, list them and say that Fire Opal tools ask which one to use, or take `organization`.

2. **Hardware credentials.** Call `qctrl_list_credentials` and list each credential's name, provider and expiry.

   - If there are none, or the user wants another provider, call `qctrl_add_credential`. The user enters the credential on a secure Q-CTRL page, not in the chat. Never ask the user to paste a token or key into the chat.
   - IBM Cloud needs an IBM Cloud API key and the instance CRN (<https://quantum.cloud.ibm.com/instances>).
   - Amazon Braket needs an IAM role ARN that trusts Q-CTRL's AWS account.
   - IonQ needs an IonQ API key. Through this server, IonQ runs only the QAOA solver.
   - To replace a credential, call `qctrl_add_credential` and save with the same name. To remove one, confirm the names with the user, then call `qctrl_remove_credentials`. Removing cannot be undone.

3. Report a short checklist: signed in, organization with Fire Opal access, and credentials per provider.

4. When everything is ready, suggest `/fire-opal:devices` as the next step.
