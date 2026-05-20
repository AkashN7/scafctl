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
7. [Command Behavior](#command-behavior)
8. [Concerns](#concerns)
9. [Future Enhancements](#future-enhancements)

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
          - provider: static
            inputs:
              value: "us-east-1"
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
          - provider: static
            inputs:
              value: "us-east-1"
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
          - provider: static
            inputs:
              value: "us-east-1"
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

---

## Command Behavior

State behavior varies across the three commands that support it. Understanding these differences is important for building correct stateful solutions.

### `run resolver`

Loads state before resolvers execute and **saves state immediately after resolvers complete**.

- State is loaded before resolver execution
- Resolvers with `saveToState: true` have their values persisted after all resolvers succeed
- If any resolver fails, state is NOT saved (no partial state)
- The `command.subcommand` field in state is recorded as `"run resolver"`

This is the simplest state lifecycle -- load, resolve, save.

### `run solution` and `run action`

Loads state before resolvers execute but **saves state only after actions complete successfully**.

- State is loaded before resolver execution (same as `run resolver`)
- Resolvers execute and produce values
- Actions execute using resolver data
- State is saved only after successful action execution
- If actions fail, state is NOT saved -- even if resolvers succeeded
- `run action` filters to specific actions (plus transitive dependencies) but follows the same state lifecycle as `run solution`
- The `command.subcommand` field is recorded as `"run solution"` or `"run action"`

This means: if you run a solution with actions and an action fails, the resolver values are not persisted to state. This is intentional -- it ensures state reflects only fully successful executions.

### `render solution`

Loads state (read-only) but **never saves state**.

- State is loaded before resolver execution
- The state provider can read previously cached values
- Resolvers execute using state context
- The action graph is rendered (not executed) using resolved values
- State is NEVER written -- render is a read-only operation
- The `command.subcommand` field passed to state load is `"render solution"`

Use `render solution` to preview what an action graph would look like with current state values, without modifying state.

### Summary Table

| Command | Loads State | Saves State | Save Trigger |
|---------|-------------|-------------|--------------|
| `run resolver` | Yes | Yes | After resolvers complete |
| `run solution` | Yes | Yes | After actions succeed |
| `run action` | Yes | Yes | After actions succeed |
| `render solution` | Yes | No (read-only) | -- |

### Resolver Chain Pattern for Parameter Override

When you want CLI parameters to override cached state values, place the `parameter` provider **before** the `state` provider in the resolve chain:

```yaml
resolvers:
  environment:
    type: string
    saveToState: true
    resolve:
      with:
        - provider: parameter      # wins if -r env=value is passed
          inputs:
            key: "env"
        - provider: state           # wins on repeat runs (no param)
          inputs:
            key: "environment"
            required: true
        - provider: static          # default on first run (no state)
          inputs:
            value: "dev"
```

The fallback chain uses `onError: continue` (the default) so each provider that fails simply falls through to the next.

---

## Concerns

### Replay and Validation Workflows

A common pattern in scaffolding frameworks is a **validation layer** that replays a previously executed solution to verify that the generated output matches what a user committed. For example, a CI validator might:

1. Read the state file from a pull request
2. Re-run the solution using that state
3. Verify the generated files match the PR contents

This pattern works because the resolver fallback chain reads cached values from `state.values` without needing any CLI parameters. However, there are important limitations and design considerations.

### The `command.parameters` Field Is Informational Only

The `command` field in state (`command.subcommand` and `command.parameters`) records the most recent invocation's CLI parameters. However:

- It is **never read back** by scafctl during execution -- it is purely metadata
- It uses **last-write-wins** semantics -- only the most recent invocation is stored, with no history
- Parameters are **string-coerced** via `fmt.Sprintf("%v", v)`, which is lossy for complex types
- Running different commands (e.g., `run resolver` then `run action`) overwrites the record entirely

This means `command.parameters` cannot be used as a reliable replay mechanism. The actual "replay" in scafctl happens through `state.values` and the resolver fallback chain.

### State Completeness for Replay

For replay to produce deterministic output, **every resolver whose value affects the generated files** must either:

1. Be saved to state (`saveToState: true`), OR
2. Be deterministically derivable from other saved resolvers (computed/transformed values)

Resolvers that are NOT saved to state will have no value during replay unless they can resolve from a non-interactive provider (e.g., `static`, `exec`, or another deterministic source). If a resolver falls through to `parameter` and no parameter is provided, it will fail.

This creates a tension: some values may be intentionally re-prompted for safety (e.g., target environment confirmation), but excluding them from state breaks deterministic replay.

**Guideline**: For solutions that require validation replay, all resolvers contributing to output must be `saveToState: true`. Safety concerns (like confirming dangerous operations) should be handled through separate mechanisms -- `when` conditions on actions, lint rules, or CI policy checks -- rather than by omitting values from state.

### Derived Values Are Safe to Exclude

Resolvers whose values are purely computed from other resolvers do not need `saveToState: true`:

```yaml
resolvers:
  base_url:
    type: string
    saveToState: true
    resolve:
      with:
        - provider: state
          inputs: { key: "base_url", required: false }
        - provider: parameter
          inputs: { key: "base_url" }

  api_endpoint:
    # Derived from base_url -- no need to save
    type: string
    resolve:
      with:
        - provider: static
          inputs:
            value:
              rslvr: base_url
    transform:
      with:
        - provider: cel
          inputs:
            expression: '__self + "/api/v2"'
```

On replay, `base_url` comes from state, and `api_endpoint` is recomputed identically.

### Volatile Values and Replay

Values that change between runs (auth tokens, timestamps, git branch) pose a challenge:

- If saved to state, replay uses the stale cached value (which may be what you want for determinism)
- If not saved, replay must obtain a fresh value (which may differ from the original run)

For validation workflows, stale-but-deterministic is usually preferable to fresh-but-different.

---

## Future Enhancements

The following are potential improvements to address the concerns above. These are not yet implemented.

### Replay Command

A dedicated `scafctl state replay` command that re-executes a solution using only state values, failing fast if any resolver cannot resolve without interactive input. This would formalize the validation workflow:

```bash
# Hypothetical: replay using state file, output to temp dir for diffing
scafctl state replay -f app-registration.yaml --state-path ./state.json --output-dir /tmp/validate
```

### Parameter Accumulation in State

Instead of last-write-wins for `command.parameters`, accumulate all parameters ever passed across runs. This would make the command field a more complete audit trail:

```json
{
  "command": {
    "subcommand": "run solution",
    "parameters": {
      "app_name": "my-app",
      "admin_group": "new-admins",
      "region": "us-east-1"
    },
    "parameterHistory": [
      {"timestamp": "2026-01-15T10:00:00Z", "parameters": {"app_name": "my-app", "region": "us-east-1"}},
      {"timestamp": "2026-03-20T14:30:00Z", "parameters": {"admin_group": "new-admins"}}
    ]
  }
}
```

### State Completeness Lint Rule

A lint rule that warns when a solution has `state.enabled: true` but has resolvers without `saveToState: true` that are not provably derived from other saved resolvers. This would catch replay gaps at authoring time rather than at validation time.

### Explicit State File Path Override

A CLI flag to point the state backend at an arbitrary file path, making it easier for validators to use a state file from a PR checkout without copying it to the XDG state directory:

```bash
# Hypothetical: override state path for validation
scafctl run solution -f app-registration.yaml --state-file ./apps/my-app/state.json
```

### Resolver Tagging for Replay Scope

A mechanism to tag resolvers as "replay-required" vs "ephemeral", allowing the replay command to distinguish between resolvers that must come from state and resolvers that are expected to be re-derived:

```yaml
resolvers:
  admin_group:
    saveToState: true
    replayScope: required    # must be in state for replay to succeed

  auth_token:
    saveToState: false
    replayScope: ephemeral   # re-derived on every run, not needed for replay
```

### Render-Based Validation Mode

A `render solution` enhancement that outputs the file tree that actions *would* produce (without executing them), enabling validators to diff against PR contents without any side effects:

```bash
# Hypothetical: render file tree to directory for comparison
scafctl render solution -f app-registration.yaml --output-dir /tmp/validate --state-file ./state.json
```
