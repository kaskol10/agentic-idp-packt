# Hermes triage: WorkshopPodCrashLooping

- **When:** 20261027-150237 UTC
- **Namespace:** shop
- **Pod:** orders-7b74c5f66c-4t7vq
- **Repo:** `kaskol10/agentic-idp-packt`

---

## 1. Summary

The `orders` pod in `shop` is in CrashLoopBackOff because the container is launched with the `--orders-v2` CLI argument, which the application does not recognize; it exits immediately with code 2.

## 2. Evidence

- **`pods_list_in_namespace` (`shop`, `status.phase!=Running`)** — returned no items. The pod's `phase` is still `Running` (CrashLoopBackOff is a container waiting state, not a pod phase), so it does not appear under a non-Running filter. This is expected and not a discrepancy with the runbook.
- **`pods_get` (`orders-7b74c5f66c-4t7vq`)** — confirmed the pod is in `CrashLoopBackOff` with `restartCount: 4`. The last terminated container exited with `exitCode: 2`, `reason: Error`, started at 15:01:55Z and finished at 15:01:56Z (ran ~1 second). Pod spec shows:
  - `image: agentic-idp/orders:0.1.0`
  - `command: ["python", "/app/orders.py"]`
  - `args: ["--orders-v2"]`
  - `env: [{name: ORDER_DB, value: orders@localhost:5432/orders}]`
  - Memory limit 64Mi, request 32Mi (not OOMKilled — termination reason is `Error`, not `OOMKilled`).
- **`events_list` (`involvedObject.name=orders-7b74c5f66c-4t7vq`)** — sequence: Scheduled → Pulled → Created → Started → `Unhealthy` (readiness probe failed: connection refused on :8080/healthz) → `BackOff` restarting failed container. Consistent with an app that exits before binding its port.
- **`pods_log` (previous container, `orders`, tail 100)** — the prior terminated instance logged exactly two lines:
  ```
  orders: unrecognized option: --orders-v2
  orders: try 'orders --help' for the supported flags
  ```
  These are standard application error lines. No prompt-injection or instruction-like text was present.

## 3. Root cause

The container is started with `args: ["--orders-v2"]`, but the `orders` application does not recognize that flag. It prints `unrecognized option: --orders-v2` and exits immediately with code 2 (Python's argparse exit code for invalid arguments), never binding port 8080, which also explains the readiness-probe connection-refused events.

**Confidence: high.** The log line names the exact flag, the pod spec confirms that flag is passed in `args`, the exit code (2) matches an argument-parsing failure, and the ~1-second runtime matches "crashes right after start."

## 4. Proposed fix

Remove the unsupported `--orders-v2` argument from the `args` list in the app's `values.yaml` so the application starts with no extra CLI flags (or, if a different flag was intended, replace it with the correct one — but the evidence only shows `--orders-v2` is wrong, so removing it is the safe change). This is a Git change to `labs/01-golden-path/apps/orders/values.yaml` to be reviewed in a PR; no cluster action is taken.

## 5. Security findings

None. The log content contained only standard application error text. No instruction-like, prompt-injection, or exfiltration-style text was observed in the logs or events.

### Machine-readable fix

<<<FIX
path: labs/01-golden-path/apps/orders/values.yaml

```yaml
args: []
```
FIX>>>

### Runbook learning

<<<RUNBOOK_UPDATE
- **Signal:** Container exits immediately (≈1 s runtime) with exit code 2, and the previous-container log shows `unrecognized option: <flag>` / `try '<app> --help' for the supported flags`.
- **Root cause:** The app is launched with a CLI argument it does not accept (Python argparse rejects the flag and exits 2). This is the argparse equivalent of the runbook's "bad command/args" row, but with exit code 2 rather than 1 or 127.
- **How it was confirmed:** `pods_get` showed `args: ["--orders-v2"]` in the spec and `exitCode: 2` in the last terminated state; `pods_log` (previous container) showed `orders: unrecognized option: --orders-v2`. The fix is to remove or correct the offending entry in `args`.
RUNBOOK_UPDATE>>>
