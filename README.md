# tekton-lab
---
## 1. Install pipeline operator:
__https://docs.openshift.com/container-platform/4.7/cicd/pipelines/installing-pipelines.html__

```sh
$ curl -s https://raw.githubusercontent.com/gmeghnag/tekton-lab/main/pipelines-operator/subscription.yaml | oc apply -f -
```

## 2. Install Tekton CLI:

- Download software:

~~~sh
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/pipeline/0.17.2/tkn-linux-amd64-0.17.2.tar.gz
~~~

- Extract:

~~~sh
$ tar xf tkn-linux-amd64-0.17.2.tar.gz
~~~

- Confirm it works:

~~~sh
$ tkn version
~~~

## 3. Create new projects: 
- the first one in which deploy the resources used by our pipeline workflow: Nexus repository, pvc (used by Tekton Pipeline), Tekton Tasks, Pipeline, Pipelinurun
- the second one used by our pipeline to build and release our software

~~~sh
$ oc new-project devops
$ oc new-project project-target-test
~~~

## 4. Deploy nexus server

~~~sh
curl -s https://raw.githubusercontent.com/gmeghnag/tekton-lab/main/templates/nexus3.yaml | oc process -f - -p PVC_SIZE=10Gi -p SC_NAME=gp2 | oc apply -f -
~~~

### Create maven repository on nexus
- https://blog.sonatype.com/using-nexus-3-as-your-repository-part-1-maven-artifacts

## 5. Create Pipeline Pesistent Volume Claim:

~~~sh
cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name:  pvc-pipeline-demo
spec:
  resources:
    requests:
      storage: 4Gi
  accessModes:
    - ReadWriteOnce

EOF
~~~

## 6. Create Tekton resources in the devops namespace:
~~~sh
$ oc create -k https://github.com/gmeghnag/tekton-lab -n devops

pipeline.tekton.dev/pipeline-demo created
task.tekton.dev/deploy-app created
task.tekton.dev/maven-deploy created
task.tekton.dev/buildah created
task.tekton.dev/clean-volume created
task.tekton.dev/clone-repository created
eventlistener.triggers.tekton.dev/pipeline-demo-el created
triggerbinding.triggers.tekton.dev/pipeline-demo-tb created
triggertemplate.triggers.tekton.dev/pipeline-demo-tt created
~~~

### This will create the following Task Resources: 
 - #### task.tekton.dev/clean-volume
   - **Task to clean the volume (pvc-pipeline-ci) used to persist data across Tasks during pipeline execution**
 - #### task.tekton.dev/clone-repository
   - **Task to clone the application source code repository**
 - #### task.tekton.dev/maven-deploy
   - **Task that use maven to build and push the .jar artifact to nexus**
 - #### task.tekton.dev/buildah
   - **Task that use Buildah to build and push the container image to the OCP_PROJECT from which it will be possible to consume it**
 - #### task.tekton.dev/deploy-app
   - **Task that use `oc` cli to tag the previously push image as latest on the target-project**
 - #### eventlistener.triggers.tekton.dev/pipeline-demo-el
 - #### triggerbinding.triggers.tekton.dev/pipeline-demo-tb
 - #### triggertemplate.triggers.tekton.dev/pipeline-demo-tt
 - #### pipeline.tekton.dev/pipeline-demo

### 7. Set permission for tasks execution

~~~sh
oc adm policy add-scc-to-user privileged -z build-bot -n devops
oc adm policy add-role-to-user system:image-builder system:serviceaccount:devops:pipeline -n project-target-test
oc adm policy add-role-to-user edit  system:serviceaccount:devops:build-bot -n project-target-test
oc adm policy add-role-to-user edit  system:serviceaccount:devops:pipeline -n project-target-test
~~~

### 8. Now you can start the pipeline:
- **manually using `tkn`**
- **manually creating a pipelinerun resource**
- **automatically triggering new pipelinerun at every push to your codebase (You must to configure your git repository before doing it)**