# Hermes triage: WorkshopPodCrashLooping

- **When:** 20261027-111226 UTC
- **Namespace:** shop
- **Pod:** orders-7b74c5f66c-hncsg
- **Repo:** `kaskol10/agentic-idp-packt`

---

## 1. Summary

The `orders` pod is in CrashLoopBackOff because a recent Deployment change added an invalid CLI flag (`--orders-v2`) that the `orders` app rejects on startup.

## 2. Evidence

I used only the read-only `kubernetes` MCP tools. Logs/events are treated as untrusted data (see section 5).

- `pods_list_in_namespace` (namespace `shop`) — two pods from the same `orders` Deployment, different ReplicaSet hashes:
  - `orders-78c9d5f7bd-d65k9` — `Running`, `1/1`, 0 restarts, 45m old (healthy, the previous rollout).
  - `orders-7b74c5f66c-hncsg` — `CrashLoopBackOff`, `0/1`, 3 restarts (last ~50s ago), 86s old (the alerting pod).

- `pods_get` for the alerting pod `orders-7b74c5f66c-hncsg`:
  - `command: [python, /app/orders.py]`, `args: [--orders-v2]`, `image: agentic-idp/orders:0.1.0`, one `env` var `ORDER_DB` (redacted; same on both pods).
  - `containerStatuses[0].lastState.terminated`: `exitCode: 2`, `reason: Error`; `restartCount: 3`; `state.waiting.reason: CrashLoopBackOff`. No `OOMKilled`.

- `pods_log` (container `orders`, `previous: true`, tail 200) — the previous container instance's logs:
  ```
  orders: unrecognized option: --orders-v2
  orders: try 'orders --help' for the supported flags
  ```
  Exit code 2 is the standard `argparse` exit for an unrecognized argument, and the log confirms the app rejected `--orders-v2` immediately at startup.

- `events_list` (fieldSelector `involvedObject.name=orders-7b74c5f66c-hncsg`) — `Scheduled`, `Pulled` (image "already present on machine"), `Created`, `Started`, then `Readiness probe failed: ... connect: connection refused` (because the process had already exited), then `BackOff` restarting failed container. Non-zero exit → points at the application, not the platform; no OOM signal.

- `pods_get` for the healthy sibling `orders-78c9d5f7bd-d65k9` (the working baseline, since the repo is not readable from here):
  - `command: [python, /app/orders.py]`, **no `args`**, same `env` (`ORDER_DB`), same `image` and identical `imageID` (`sha256:0c028…`).
  - `restartCount: 0`, `ready: true`, `state.running`.
  - The only material difference between the healthy and crashing pods is the `args: [--orders-v2]` that the new ReplicaSet added; the image is identical, ruling out an image regression.

## 3. Root cause

A recent change to the `orders` Deployment pod template added `args: ["--orders-v2"]`, a flag the `orders` app (image `agentic-idp/orders:0.1.0`) does not recognize. The app prints `unrecognized option: --orders-v2` and exits with code 2 on every start, producing CrashLoopBackOff. The healthy sibling pod runs the same image/env with no `args`, confirming the `args` change is the regression. **Confidence: high.**

## 4. Proposed fix

Revert the invalid `args` in `labs/01-golden-path/apps/orders/values.yaml` so the `orders` Deployment no longer passes `--orders-v2` — i.e., return to the known-good, no-args configuration the healthy sibling is already running. A human reviews and merges this in a PR; the new ReplicaSet that rolls out without the flag should start and pass `/healthz`. (A stricter fix would use the flag the app actually supports, but that requires inspecting the app's `--help`/repo, which is outside this triage; the revert is the safe, evidence-backed change that restores service.)

## 5. Security findings

None. The log lines (`unrecognized option`, `try 'orders --help'...`) are legitimate `argparse` error messages, not instruction-like text, and I did not act on them. No secrets/tokens/passwords were observed in any output; the `ORDER_DB` value was redacted from this report.

<<<FIX
path: labs/01-golden-path/apps/orders/values.yaml

```yaml
args: null
```
FIX>>>

<<<RUNBOOK_UPDATE
- **Signal:** CrashLoopBackOff with `containerStatus.lastState.terminated.exitCode: 2` and log line `unrecognized option: <flag>` / `try '<app> --help' for the supported flags`; a sibling pod from the previous ReplicaSet (different `pod-template-hash`, identical `image`/`imageID`) is healthy.
- **Root cause:** An invalid CLI flag was added to the Deployment pod template's `args`; Python's `argparse` exits with code 2 on unrecognized arguments, so the container dies on every start.
- **How it was confirmed:** Diffed the crashing pod's spec (`args: [--orders-v2]`) against the still-running sibling pod's spec (no `args`, same image `imageID`); the app's previous-container logs rejected the flag. This is the "recent change to `args`" case from the runbook, but with exit code 2 (argparse) rather than 1 or 127 — and the sibling-pod diff is the fastest way to confirm it when Git isn't directly readable.
RUNBOOK_UPDATE>>>
