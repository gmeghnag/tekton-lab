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

## 3. Create new project in which deploy the resources used by our pipeline workflow (Nexus repository, pvc (used by Tekton Pipeline), Tekton Tasks, Pipeline, Pipelinurun):

~~~sh
$ oc new-project devops
~~~

## 4. Deploy nexus server

~~~sh
cat << EOF | oc process -f - | oc apply -f -
apiVersion: v1
kind: Template
metadata:
  name: "Nexus"
  annotations:
    openshift.io/display-name: "Nexus application"
    description: "Nexus artifact Repository Template"
    tags: "nexus, devops"
objects:
- kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: nexus3-pvc
    labels:
      app: nexus3
  spec:
    storageClassName: gp2
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
- kind: DeploymentConfig
  apiVersion: apps.openshift.io/v1
  metadata:
    labels:
      app: nexus3
    name: nexus3
  spec:
    volumes:
    - name: nexus3-volume
      persistentVolumeClaim:
        claimName: nexus3-pvc
    replicas: 1
    selector:
      deploymentconfig: nexus3
    strategy:
      type: Rolling
    template:
      metadata:
        labels:
          deploymentconfig: nexus3
      spec:
        containers:
          - name: nexus3
            image: sonatype/nexus3
            env:
            - name: INSTALL4J_ADD_VM_PARAMS
              value: -XX:ActiveProcessorCount=1
            ports:
              - containerPort: 8081
                protocol: TCP
            resources: {}
            volumeMounts:
              - name: nexus3-volume
                mountPath: /nexus-data
        volumes:
        - name: nexus3-volume
          persistentVolumeClaim:
            claimName: nexus3-pvc
        restartPolicy: Always
- apiVersion: v1
  kind: Service
  metadata:
    name: nexus3
    labels:
      app: nexus3
  spec:
    ports:
    - port: 8081
      protocol: TCP
      targetPort: 8081
      name: 8081-tcp
    selector:
      deploymentconfig: nexus3
    sessionAffinity: None
    type: ClusterIP
  status:
    loadBalancer: {}
- kind: Route
  apiVersion: v1
  metadata:
    name: nexus3
  spec:
    port:
      targetPort: 8081-tcp
    to:
      kind: Service
      name: nexus3
EOF
~~~

###Create nexus repository - maven
- https://blog.sonatype.com/using-nexus-3-as-your-repository-part-1-maven-artifacts

## 5. Create Pipeline Pesistent Volume Claim:

~~~sh
cat << EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name:  pvc-pipeline-ci
spec:
  resources:
    requests:
      storage: 10Gi
  accessModes:
    - ReadWriteOnce

EOF
~~~

## 6. Create tasks resources in the namespace:
~~~sh
$ oc apply -k https://github.com/gmeghnag/tekton-lab -n devops

task.tekton.dev/clean-volume created
task.tekton.dev/clone-repository created
task.tekton.dev/maven-deploy created
task.tekton.dev/buildah created
task.tekton.dev/deploy-app created
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

### 7. Create the pipeline

~~~sh
cat << EOF | oc apply -f -
kind: Pipeline
apiVersion: tekton.dev/v1alpha1
metadata:
  name: pipeline-ci
spec:
  params:
    - name: REPOSITORY_URL
      type: string
    - name: REGISTRY_URL
      type: string
    - name: BUILD_IMAGE
      type: string
    - name: OCP_PROJECT
      type: string
    - name: BRANCH_NAME
      type: string
  workspaces:
    - name: TKN_VOLUME
  tasks:
    - name: clean-volume
      taskRef:
        name: clean-volume
      workspaces:
        - name: VOLUME
          workspace: TKN_VOLUME
    - name: clone-repository
      runAfter:
        - clean-volume
      params:
        - name: REPOSITORY_URL
          value: "$(params.REPOSITORY_URL)"
        - name: BRANCH_NAME
          value: "$(params.BRANCH_NAME)"
      taskRef:
        name: clone-repository
      workspaces:
        - name: VOLUME
          workspace: TKN_VOLUME
    - name: maven-deploy
      runAfter:
        - clone-repository
      taskRef:
        name: maven-deploy
      workspaces:
        - name: VOLUME
          workspace: TKN_VOLUME
    - name: build-image
      runAfter:
      - maven-deploy
      taskRef:
        name: buildah
      workspaces:
        - name: VOLUME
          workspace: TKN_VOLUME
      params:
      - name: REGISTRY_URL
        value: "$(params.REGISTRY_URL)"
      - name: BUILD_IMAGE
        value: "$(params.BUILD_IMAGE)"
      - name: OCP_PROJECT
        value: "$(params.OCP_PROJECT)"
    - name: deploy-app
      runAfter:
      - build-image
      taskRef:
        name: deploy-app
      workspaces:
        - name: VOLUME
          workspace: TKN_VOLUME
      params:
      - name: BUILD_IMAGE
        value: "$(params.BUILD_IMAGE)"
      - name: OCP_PROJECT
        value: "$(params.OCP_PROJECT)"
EOF
~~~

### 7. Set permission for tasks execution

~~~sh
oc policy add-scc-to-user privileged -z build-bot -n devops
oc policy add-role-to-user system:image-builder system:serviceaccount:devops:pipeline -n project-target-test
oc policy add-role-to-user edit  system:serviceaccount:devops:pipeline -n project-target-test
~~~


### 8. Start the pipeline

~~~sh
cat << EOF > | oc apply -f -
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  generateName: springapp-deploy-run-
  labels:
    tekton.dev/pipeline: pipeline-ci
spec:
  serviceAccountName: pipeline
  serviceAccountNames:
    - taskName: build-image
      serviceAccountName: build-bot
  pipelineRef:
    name: pipeline-ci
  params:
    - name: "REPOSITORY_URL"
      value: "https://github.com/GMH501/spring-petclinic.git"
    - name: REGISTRY_URL
      value: image-registry.openshift-image-registry.svc:5000
    - name: "BUILD_IMAGE"
      value: "spring-petclinic"
    - name: "OCP_PROJECT"
      value: "project-target-test"
    - name: "BRANCH_NAME"
      value: "main"
  workspaces:
  - name: TKN_VOLUME
    persistentVolumeClaim:
      claimName: pvc-pipeline-ci
EOF
~~~

