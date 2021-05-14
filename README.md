# tekton-lab
---
## 1. Install pipeline operator:
__https://docs.openshift.com/container-platform/4.7/cicd/pipelines/installing-pipelines.html__

```sh
sh curl -s https://raw.githubusercontent.com/gmeghnag/tekton-lab/main/pipelines-operator/subscription.yaml | oc apply -f -
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

## 3. Create new project in which deploy the resources used by our pipeline workflow:

~~~sh
$ oc new-project devops
~~~

## 4. 

~~~sh
tkn pipeline start greetings -p FIRST_GREETING=REDHATTERS -p SECOND_GREETING=WORLD --showlog
~~~

## 5. Create tasks resources in the namespace:
~~~sh
$ oc apply -k https://github.com/gmeghnag/tekton-lab -n devops

task.tekton.dev/clean-volume created
task.tekton.dev/clone-repository created
task.tekton.dev/maven-deploy created
task.tekton.dev/buildah created
task.tekton.dev/deploy-app created
~~~