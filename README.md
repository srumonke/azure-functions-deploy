# Azure Function Deployment Without Kubernetes

A full CD Deploy stage (service, environment, infrastructure, rollback) for an Azure Function
that runs entirely as shell on the delegate. No containerized step group. No Kubernetes
connector anywhere in the pipeline.

## Status: verified green

| | |
|---|---|
| Pipeline | `azure_function_shell_demo` |
| Verified run | run 1, execution `6U6x8mqvQ465ytYP1PXPBA` — **Success in 41s** |
| Kubernetes used | none |
| Azure subscription needed | none (see *Simulated mode* below) |

## Approach

There is no shell variant of the native `AzureFunctionDeploy` step, but the deployment itself
is just Azure CLI calls. Those are wired into a Custom Deployment Template, which is the
sanctioned way to model a deployment Harness has no containerless native support for. The
whole CD model is preserved; the only thing given up is the plugin's built-in rollback, which
is replaced here by one explicit swap-back step.

## What was created

| Resource | Identifier | Notes |
|---|---|---|
| Deployment Template | `azure_function_shell_deploy` (v1) | `CustomDeployment`; infra vars + fetch-instances script |
| Service | `azure_function_demo` | `CustomDeployment`, no artifact — standalone |
| Infrastructure Definition | `azure_function_test` | env `test`, resource group + subscription |
| Pipeline | `azure_function_shell_demo` | 6 steps + 1 rollback step, all `ShellScript` |


## Pipeline flow

```
Show Execution Context   -> prints hostname/user; proves it's the delegate, not a pod
Fetch Instances          -> deployment template publishes the slots as CD instances
Deploy To Staging Slot   -> az functionapp deployment source config-zip  (or container set)
Health Check Staging     -> gate BEFORE any production traffic moves
Swap Slot To Production  -> az functionapp deployment slot swap  (zero-downtime cutover)
Verify Production        -> non-200 here triggers the rollback

Rollback: Swap Back      -> reverse swap; old version is still sitting on the staging slot
```

The slot-swap pattern gives the same zero-downtime behaviour the native plugin provides, so
functionally very little is given up.

## Simulated mode

Every script probes for `az` and branches:

```bash
if command -v az >/dev/null 2>&1; then SIM=0; else SIM=1; fi
run() { if [ "$SIM" -eq 1 ]; then echo "  [SIMULATED] $*"; else echo "  [RUN] $*"; "$@"; fi; }
```

The account's delegates are Linux pods without the Azure CLI, so it runs in SIMULATED mode: it
prints the exact `az` command it *would* run, then continues. That is what makes the pipeline
green with no Azure account — and it is also the proof that simulated mode engaged, since a
real `az login` with the placeholder secret would have failed the step.

The execution logs show the real commands, in order, with real resolved values.

## Making it live

1. **Install the Azure CLI on the delegate.** This is the single most common failure
   (`az: command not found`) — use delegate custom binaries / `INIT_SCRIPT`, or a custom
   delegate image. `SIM` flips to `0` automatically; no YAML change.
2. **Create a secret** `azure_sp_secret` (service principal password) and replace the three
   `REPLACE_WITH_HARNESS_SECRET` literals with `<+secrets.getValue("azure_sp_secret")>`.
3. **Set the real values** in `.harness/infrastructure.yaml` (`resourceGroup`,
   `subscriptionId`) and `.harness/service.yaml` (`functionApp`, `artifactUrl` /
   `containerImage`).
4. **Add a real artifact source** to the service if the package should come from JFrog/ACR
   rather than a URL.
5. **Add a production site**: copy the infra definition block, change `environmentRef` to
   `production`. Template, service and pipeline are reused unchanged.

## Files

```
azure functions/
├── README.md
├── demo.md
└── .harness/
    ├── deployment-template.yaml   # CustomDeployment template (the key piece)
    ├── service.yaml
    ├── infrastructure.yaml
    └── pipeline.yaml
```

These mirror what is live in Harness. Editing a file does not update Harness — the entities are
inline, not Git-backed.
