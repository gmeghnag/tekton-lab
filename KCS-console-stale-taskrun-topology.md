# Title
OpenShift Pipelines console displays a PipelineRun task as failed when a PipelineRun with the same name was previously deleted in RHOCP 4

# Issue
- In the OpenShift Pipelines console, the PipelineRun details page (topology) shows a task with a red "!" icon and a step count of `0/1`, indicating that the task failed.
- The TaskRun for that task actually succeeded: `oc get taskrun` reports `Succeeded=True`.
- The PipelineRun itself completed with status `Succeeded`.
- The problem appears after a previous PipelineRun with the same name was deleted from the cluster.
- Only the task whose name matches the previous run is displayed incorrectly; the other task nodes in the topology are correct.

# Environment
- Red Hat OpenShift Container Platform (RHOCP)
  - 4.21
- Red Hat OpenShift Pipelines
  - 1.24
- Tekton Results enabled (deployment `tekton-results-api` in the namespace `openshift-pipelines`)

# Resolution
The pipeline execution itself is not affected; only the console display is wrong. Until a fix is released, apply one of the following workarounds:
1. workaround: do not reuse the same PipelineRun name. Create each PipelineRun with a unique name, for example by appending a timestamp.
2. workaround: if the same name must be reused, delete the stale TaskRun record from Tekton Results before creating the new PipelineRun.

A bug has been reported to Red Hat for this display issue. A fix is expected in a future release of Red Hat OpenShift Pipelines; upgrade to that release when available.

# Root Cause
The OpenShift Pipelines console builds the PipelineRun topology by merging TaskRun data from the cluster API and from the Tekton Results archive. The Results query filters by PipelineRun name only, so when a previously deleted PipelineRun left a stale TaskRun record with the same task name, the console displays the state of the stale record instead of the state of the current TaskRun.

# Diagnostic Steps
1. Verify the actual state of the TaskRun:
   ```
   oc get taskrun <taskrun-name> -n <namespace> \
     -o jsonpath='{.status.conditions[0].status} {.status.conditions[0].reason}'
   ```
   Output:
   ```
   True Succeeded
   ```
2. Verify the PipelineRun state:
   ```
   oc get pipelinerun <pipelinerun-name> -n <namespace> \
     -o jsonpath='{.status.conditions[0].status} {.status.conditions[0].reason}'
   ```
   Output:
   ```
   True Succeeded
   ```
3. Confirm that the console topology node for the task shows the red "!" icon and the `0/1` badge while the API reports success.
4. Confirm that Tekton Results is enabled and that a previous PipelineRun with the same name was deleted, which leaves a stale TaskRun record in the Results archive.

# Tags
shift_pipelines, ocp_4

# Internal Tags
openshift-pipelines, tekton-results, ocp_4
