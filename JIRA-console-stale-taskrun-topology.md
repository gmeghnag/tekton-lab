# [Console] PipelineRun topology shows a task as **failed** when a stale same-name TaskRun from a previous PipelineRun is still in Tekton Results

## Versions
| Component | Version |
|---|---|
| OpenShift Container Platform (OCP) | 4.21.37 (K8s v1.34.9) |
| OpenShift Pipelines operator | **1.24.3 (latest) — bug present** |
| Tekton Pipelines | v1.15.5 |
| Tekton Results | enabled (`pipelines-results-api-rhel9`) |
| Console plugin | `pipelines-console-plugin-pf5-rhel9` (`@33cb80d0…`) |

**Status (2026-11-11):** tested on the **latest** channel `pipelines-1.24` (CSV
`openshift-pipelines-operator-rh.v1.24.3`) — the bug **reproduces** (topology node
`pf-m-danger` + `0/1`, K8s TaskRun `Succeeded`), i.e. **not fixed as of 1.24.3**. Confirmed
in code on `release-v1.24.x` (and `main`/`release-v1.25.x`): `PipelineRunVisualization.tsx`
still calls `useTaskRuns(ns, name, undefined, undefined, …)` — `pipelineRunUid` is **not**
passed, so the Tekton Results query is not narrowed to the current run's UID and the stale
failed record wins.

## Description
When a PipelineRun is created with the **same name** as a previously deleted PipelineRun, the task node in the PipelineRun detail **topology** can show a **failed** icon (red "!" / `ExclamationCircle`) and a wrong step count (e.g. `0/1`), even though the **current** TaskRun for that task **succeeded** (step `exitCode=0`, condition `Succeeded=True`).

The console plugin merges TaskRuns from the Kubernetes API with those from the **Tekton Results API** and ends up rendering the **stale (failed) record of the previous run** instead of the current (succeeded) one.

## Expected result
The task node reflects the state of the **current** TaskRun for that PipelineRun, consistent with the API:
- `oc get taskrun <plr>-maven-deploy -n devops` → `Succeeded=True`, step `exitCode=0`
- Topology node → green `CheckCircle`, badge `1/1`

## Actual result
- Topology node for `maven-deploy` → red `pf-m-danger` (`ExclamationCircle`), badge `0/1`.
- All 5 TaskRuns of the current run are `Succeeded` in the API (the PR itself is `Succeeded`, `Tasks Completed: 5 (Failed: 0)`).
- The other 4 task nodes render correctly; only the task whose **same name** has a stale record in Results is wrong.

## Root cause (buggy code, repo `openshift-pipelines/console-plugin`, `release-v1.24.x` and `main`)
The plugin fetches the PipelineRun's TaskRuns from two sources and merges them:
1. **Kubernetes API** (live) — filtered by label `tekton.dev/pipelineRun=<name>`.
2. **Tekton Results API** — filtered **only** by `data.metadata.labels["tekton.dev/pipelineRun"] == "<name>"` (no UID).

When a previous PipelineRun with the same name left its TaskRun record in Results (the k8s object was deleted, the Results record persists), the Results query returns **both** the stale TaskRun (previous run) and the current one. Three code paths then pick the wrong one:

1. **Diagram does not pass the PipelineRun UID**, so the Results filter is not narrowed to this run:
   - `src/components/pipelineRuns-details/PipelineRunVisualization.tsx`
     ```js
     useTaskRuns(pipelineRun?.metadata?.namespace, pipelineRun?.metadata?.name, {
       pipelineRunFinished,
       pipelineRunManagedBy: pipelineRun?.spec?.managedBy
     });   // pipelineRunUid NOT passed
     ```
   - Upstream **PR #350** (SRVKP-7409, "Tekton Results shows TaskRuns from previous PipelineRuns of the same name") added a `pipelineRunUid` parameter to `useTaskRuns()` but wired it **only into `TaskRunsList.tsx`**, not into this topology view — so the diagram is still affected.

2. **Merge dedupes by UID, not by name**, so the stale and the current TaskRun (same `metadata.name`, different `metadata.uid`) **both survive**:
   - `src/components/hooks/useTaskRuns.ts`
     ```js
     uniqBy([...etcdRuns, ...trResources], (r) => r.metadata.uid)
     ```

3. **Task status is taken from the first match by task name**, and the list is pre-sorted by an (incorrect) ascending-completion-time comparator, so the **earlier-completed stale (failed) TaskRun comes first** and wins:
   - `src/components/utils/pipeline-utils.ts` → `appendPipelineRunStatus`
     ```js
     const taskRun = _.find(taskRuns,
       (tr) => tr.metadata.labels[TektonResourceLabel.pipelineTask] === task.name);
     ```
   → `status.reason = Failed` → `ExclamationCircle` + `0/1` step badge.

## How to reproduce
**Minimal, tested reproducer:** https://github.com/gmeghnag/stale-taskrun-repro
(`oc apply -k run1-fail` → delete → `oc apply -k run2-succeed` → inspect the console).
See the README there for the 3-command walkthrough and a screenshot of the bug.

Manual steps:
1. Cluster with OpenShift Pipelines + **Tekton Results enabled**.
2. Create a Pipeline `pipeline-demo` with at least one task (e.g. `maven-deploy`) and a PipelineRun named `pipeline-demo-run-04`.
3. Run it (task fails or completes), then **delete** the PipelineRun. The k8s objects are removed, but the TaskRun records remain in Tekton Results.
4. Create a **new** PipelineRun with the **same name** `pipeline-demo-run-04` and let `maven-deploy` **succeed**.
5. Console → `devops` → PipelineRun `pipeline-demo-run-04` → **Overview** (topology).
6. Observe the `maven-deploy` node: red "!" (`pf-m-danger`) and `0/1`, while
   `oc get taskrun pipeline-demo-run-04-maven-deploy -n devops` shows `Succeeded` / exit `0`.

## Workaround
- Do **not** reuse the same PipelineRun name (append a timestamp/UID), **or**
- Purge the stale TaskRun record from Tekton Results.

## Suggested fix
Pass `pipelineRunUid: pipelineRun.metadata.uid` to `useTaskRuns` in
`PipelineRunVisualization.tsx` (as already done in `TaskRunsList.tsx`), so the Tekton
Results query is also filtered by `tekton.dev/pipelineRunUID`.
