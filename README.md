# kluctl-1589-hook-abort-repro

Minimal reproduction for [kluctl/kluctl#1589](https://github.com/kluctl/kluctl/issues/1589):
**`--abort-on-error` does not stop applying remaining hooks after a hook failure/timeout.**

With `--abort-on-error`, the regular apply loop honours the abort signal on every iteration
(`pkg/deployment/utils/apply_utils.go:706-708`), but the hook apply/wait loop in
`RunHooks` (`pkg/deployment/utils/hooks_util.go`) checks `abortSignal` only once before
building its lists (line 72) and never inside the loop (lines 102-116). After a hook times
out and `HandleError` sets the abort signal (`apply_utils.go:166-168`), the remaining hooks
of the same batch are still applied and waited on.

## Layout

```
.kluctl.yaml                 # single target "repro" (context: kind-sakuga-first-run)
deployment.yml               # two deployment items: manifests-ns, manifests-main
manifests-ns/ns.yaml         # namespace kluctl-repro
manifests-main/hook-hang.yaml  # pre-deploy hook Job, sleeps 3600s, hook-timeout: 15s
manifests-main/hook-fast.yaml  # pre-deploy hook Job, completes instantly, hook-timeout: 30s
manifests-main/cm-after.yaml   # regular ConfigMap in the same deployment item
logs/run1-default.log        # control run without --abort-on-error (by-design continuation)
logs/run2b-abort.log         # repro run with --abort-on-error (the bug)
```

Note: `ns.yaml` is kept for completeness, but for a deterministic run the namespace is
pre-created manually before `kluctl deploy` (items within one layer are applied
concurrently with barriers, so the hook jobs can race ahead of the namespace item of the
same deployment on a fresh cluster).

## Steps to reproduce

Prerequisites: kluctl 2.28.2, any reachable cluster (tested on kind, k8s v1.37.0).

```bash
kubectl create ns kluctl-repro

# Run 1 (control): no flag — by-design best-effort continuation, rc=1 at the end
kluctl deploy -t repro --yes

# Reset state between runs (delete the configmap so its absence proves the abort):
kubectl -n kluctl-repro delete configmap cm-after

# Run 2 (the bug): --abort-on-error
kluctl deploy -t repro --yes --abort-on-error

kubectl -n kluctl-repro get jobs,cm
```

## Expected vs actual (run 2, `--abort-on-error`)

**Expected:** after `hook-hang` times out and the abort signal is set, nothing further of
the same deployment item is applied — neither `hook-fast` nor `cm-after`.

**Actual:**

- `cm-after` is **not** applied — the regular apply loop honoured the abort signal;
- `hook-fast` **is** applied, waits and completes (`job/hook-fast` = succeeded) — the hook
  loop did not check the signal.

Raw outputs: `logs/run1-default.log` (control), `logs/run2b-abort.log` (bug).

```
manifests-main: Applying hook kluctl-repro/Job/hook-hang (1 of 2)
timed out while waiting for readiness of kluctl-repro/Job/hook-hang (14s elapsed)
manifests-main: Applying hook kluctl-repro/Job/hook-fast (2 of 2)
manifests-main: Finished waiting for kluctl-repro/Job/hook-fast (3s elapsed)
manifests-main: Applying 1 objects
  kluctl-repro/Job/hook-hang: timed out while waiting for readiness of kluctl-repro/Job/hook-hang
command failed
```

## Localization (kluctl v2.28.2)

| Location | Role |
|---|---|
| `pkg/deployment/commands/deploy.go:64` | `--abort-on-error` defaults to `false` |
| `pkg/deployment/utils/apply_utils.go:487-594` | `WaitReadiness`: on timeout `status.Warningf` + `HandleError` + `return false` (lines 575-585) |
| `pkg/deployment/utils/apply_utils.go:158-172` | `HandleError`: with `AbortOnError` stores `abortSignal` (lines 166-168); readiness timeout error is a plain `fmt.Errorf`, not `context.DeadlineExceeded`, so line 162 does not trigger |
| `pkg/deployment/utils/apply_utils.go:700-708` | regular apply loop: prints `Applying N objects` **before** the loop, then checks `abortSignal` **per iteration** and breaks — this is why `cm-after` is not applied |
| `pkg/deployment/utils/hooks_util.go:67-79` | `RunHooks`: `abortSignal` checked only once, before building the apply lists (line 72) |
| `pkg/deployment/utils/hooks_util.go:102-116` | hook apply/wait loop: `ApplyObject` + `WaitReadiness` per hook, **no `abortSignal` check** — this is why `hook-fast` is still applied |
| `pkg/deployment/utils/hooks_util.go:118-135` | after the loop, wait results are consumed only for `hook-succeeded`/`hook-failed` delete policies |

Suggested fix shape: check `abortSignal` inside the `RunHooks` apply loop (per iteration,
like the regular apply loop) and skip the remaining hooks once it is set.
