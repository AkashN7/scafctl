---
title: "State Tutorial"
weight: 95
---

# State Tutorial

This tutorial walks you through using state persistence to retain resolver values across solution executions. You'll learn how to configure state, read from it on subsequent runs, and manage state files via the CLI.

## Prerequisites

- scafctl installed and available in your PATH
- Basic familiarity with YAML syntax and solution files
- Understanding of resolvers and the provider system

## Table of Contents

1. [Your First Stateful Solution](#your-first-stateful-solution)
2. [Reading from State on Subsequent Runs](#reading-from-state-on-subsequent-runs)
3. [Dynamic State Paths](#dynamic-state-paths)
4. [CLI Commands](#cli-commands)
5. [Sensitive Values](#sensitive-values)
6. [Common Patterns](#common-patterns)

---

## Your First Stateful Solution

Let's create a solution that saves resolver values to a local state file so they persist between runs.

### Step 1: Create the Solution File

Create a file called `state-demo.yaml`:

```yaml
apiVersion: scafctl.io/v1
kind: Solution
metadata:
  name: state-demo
  version: 1.0.0

state:
  enabled: true
  backend:
    provider: file
    inputs:
      path: "state-demo.json"

spec:
  resolvers:
    username:
      type: string
      saveToState: true
      resolve:
        with:
          - provider: parameter
            inputs:
              key: "username"

    region:
      type: string
      saveToState: true
      resolve:
        with:
          - provider: parameter
            inputs:
              key: "region"
              default: "us-east-1"
```

### Step 2: Run the Solution

{{< tabs "state-tutorial-cmd-1" >}}
{{% tab "Bash" %}}
```bash
scafctl run resolver -f state-demo.yaml -r username=alice -r region=eu-west-1
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl run resolver -f state-demo.yaml -r username=alice -r region=eu-west-1
```
{{% /tab %}}
{{< /tabs >}}

Output:

```
username: alice
region: eu-west-1
```

The values are now saved to `~/.local/state/scafctl/state-demo.json`.

### Understanding the Structure

- **state.enabled** -- Activates state persistence. Can be a literal `true`, a CEL expression, or template. Because state is loaded before resolvers run, resolver references (`rslvr:...`) are not supported here.
- **state.backend.provider** -- The provider that handles persistence. Use `file` for local files.
- **state.backend.inputs.path** -- Where to store the state file, relative to the XDG state directory (`~/.local/state/scafctl/` on macOS/Linux).
- **saveToState: true** -- Marks a resolver's output for persistence after execution completes.

All `saveToState` values are collected after resolvers complete, then flushed to the backend in a single save call. This ensures no partial state on failures.

---

## Reading from State on Subsequent Runs

Now let's add the `state` provider so values are read from state on subsequent runs instead of requiring parameters again.

### Step 1: Create the Solution File

Create a file called `state-fallback.yaml`:

```yaml
apiVersion: scafctl.io/v1
kind: Solution
metadata:
  name: state-fallback
  version: 1.0.0

state:
  enabled: true
  backend:
    provider: file
    inputs:
      path: "state-fallback.json"

spec:
  resolvers:
    username:
      type: string
      saveToState: true
      resolve:
        with:
          - provider: state
            inputs:
              key: "username"
              required: false
          - provider: parameter
            inputs:
              key: "username"

    region:
      type: string
      saveToState: true
      resolve:
        with:
          - provider: state
            inputs:
              key: "region"
              required: false
          - provider: parameter
            inputs:
              key: "region"
              default: "us-east-1"
```

The resolver fallback chain makes this work:

1. On first run, `state` returns `ErrKeyNotFound` (no state file exists), so the resolver falls through to `parameter` via `onError: continue`.
2. After execution, the result is saved to state via `saveToState: true`.
3. On subsequent runs, `state` returns the cached value and the chain stops.

### Step 2: First Run (Provide Parameters)

{{< tabs "state-tutorial-cmd-2" >}}
{{% tab "Bash" %}}
```bash
scafctl run resolver -f state-fallback.yaml -r username=alice -r region=eu-west-1
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl run resolver -f state-fallback.yaml -r username=alice -r region=eu-west-1
```
{{% /tab %}}
{{< /tabs >}}

Output:

```
username: alice
region: eu-west-1
```

### Step 3: Second Run (Values Come from State)

{{< tabs "state-tutorial-cmd-3" >}}
{{% tab "Bash" %}}
```bash
scafctl run resolver -f state-fallback.yaml
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl run resolver -f state-fallback.yaml
```
{{% /tab %}}
{{< /tabs >}}

Output:

```
username: alice
region: eu-west-1
```

Both values are read from state. No re-prompting needed.

### State Provider Inputs

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `key` | string | Yes | -- | State entry key (typically a resolver name) |
| `required` | bool | No | `false` | Error when key is not found |
| `fallback` | any | No | -- | Value when key is not found and `required` is `false` |

---

## Dynamic State Paths

Use Go templates in backend inputs to create per-project state files.

### Step 1: Create the Solution File

Create a file called `state-dynamic.yaml`:

```yaml
apiVersion: scafctl.io/v1
kind: Solution
metadata:
  name: state-dynamic
  version: 1.0.0

state:
  enabled: true
  backend:
    provider: file
    inputs:
      path:
        tmpl: "deploy/{{ .__params.project }}.json"

spec:
  resolvers:
    region:
      type: string
      saveToState: true
      resolve:
        with:
          - provider: state
            inputs:
              key: "region"
              required: false
          - provider: parameter
            inputs:
              key: "region"
              default: "us-east-1"
```

### Step 2: Run with Different Projects

{{< tabs "state-tutorial-cmd-4" >}}
{{% tab "Bash" %}}
```bash
scafctl run resolver -f state-dynamic.yaml -r project=frontend -r region=us-west-2
scafctl run resolver -f state-dynamic.yaml -r project=backend -r region=eu-west-1
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl run resolver -f state-dynamic.yaml -r project=frontend -r region=us-west-2
scafctl run resolver -f state-dynamic.yaml -r project=backend -r region=eu-west-1
```
{{% /tab %}}
{{< /tabs >}}

Each project gets its own state file:

```
~/.local/state/scafctl/deploy/frontend.json
~/.local/state/scafctl/deploy/backend.json
```

---

## CLI Commands

The `scafctl state` command group lets you inspect and modify state files directly.

### Step 1: List Keys

{{< tabs "state-tutorial-cmd-5" >}}
{{% tab "Bash" %}}
```bash
scafctl state list --path state-fallback.json
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl state list --path state-fallback.json
```
{{% /tab %}}
{{< /tabs >}}

### Step 2: Get a Value

{{< tabs "state-tutorial-cmd-6" >}}
{{% tab "Bash" %}}
```bash
scafctl state get --path state-fallback.json --key username
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl state get --path state-fallback.json --key username
```
{{% /tab %}}
{{< /tabs >}}

### Step 3: Set a Value Manually

{{< tabs "state-tutorial-cmd-7" >}}
{{% tab "Bash" %}}
```bash
scafctl state set --path state-fallback.json --key username --value bob
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl state set --path state-fallback.json --key username --value bob
```
{{% /tab %}}
{{< /tabs >}}

### Step 4: Delete a Key

{{< tabs "state-tutorial-cmd-8" >}}
{{% tab "Bash" %}}
```bash
scafctl state delete --path state-fallback.json --key username
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl state delete --path state-fallback.json --key username
```
{{% /tab %}}
{{< /tabs >}}

### Step 5: Clear All Values

{{< tabs "state-tutorial-cmd-9" >}}
{{% tab "Bash" %}}
```bash
scafctl state clear --path state-fallback.json
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl state clear --path state-fallback.json
```
{{% /tab %}}
{{< /tabs >}}

> [!NOTE]
> `scafctl state list` and `scafctl state get` support `-o json`, `-o yaml`, and `-o quiet` output formats. The `--path` flag is relative to the XDG state directory. Use an absolute path to reference files outside the state directory.

---

## Sensitive Values

When a resolver is marked `sensitive: true` and `saveToState: true`, the value is stored **in plaintext** in the state file. A lint warning is emitted to alert the solution author.

### Step 1: Create the Solution File

Create a file called `state-sensitive.yaml`:

```yaml
apiVersion: scafctl.io/v1
kind: Solution
metadata:
  name: state-sensitive
  version: 1.0.0

state:
  enabled: true
  backend:
    provider: file
    inputs:
      path: "state-sensitive.json"

spec:
  resolvers:
    api_key:
      type: string
      sensitive: true
      saveToState: true
      resolve:
        with:
          - provider: state
            inputs:
              key: "api_key"
              required: false
          - provider: parameter
            inputs:
              key: "API Key"
```

### Step 2: Run the Linter

{{< tabs "state-tutorial-cmd-10" >}}
{{% tab "Bash" %}}
```bash
scafctl lint -f state-sensitive.yaml
```
{{% /tab %}}
{{% tab "PowerShell" %}}
```powershell
scafctl lint -f state-sensitive.yaml
```
{{% /tab %}}
{{< /tabs >}}

Output:

```
WARNING [state-sensitive-value] Sensitive resolver "api_key" with saveToState will be stored in plaintext
```

This is an explicit, informed decision. Encryption is not used because the validation application running on a separate machine would not have access to decryption keys.

---

## Common Patterns

### Cache expensive API calls

```yaml
resolvers:
  auth_token:
    type: string
    saveToState: true
    resolve:
      with:
        - provider: state
          inputs:
            key: "auth_token"
            required: false
        - provider: http
          inputs:
            url: "https://auth.example.com/token"
            method: POST
```

On first run, the HTTP call fetches the token. On subsequent runs, it comes from state.

### Dynamic state activation

```yaml
state:
  enabled:
    expr: "__params.enable_state == true"
  backend:
    provider: file
    inputs:
      path: "my-app.json"
```

State is only active when the `enable_state` CLI parameter is set to `true` (e.g., `-r enable_state=true`).

### Writing state from actions

```yaml
workflow:
  actions:
    - name: save-deployment-id
      provider: state
      inputs:
        key: "deployment_id"
        value:
          rslvr: deployment_result
```

Actions can explicitly write values to state using the `state` provider with `action` capability.
