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

## 4. Create Task and Pipeline resources in the namespace:
~~~sh
$ git clone https://github.com/gmeghnag/tekton-lab.git -b hello-tekton
$ cd tekton-lab
$ oc create -f greet-someone.Task
$ oc create -f greetings.Pipeline
~~~

## 5. Start the task using tkn CLI

~~~sh
tkn task start greet-someone -p WHO=Redhat --showlog
tkn pipeline start greetings -p FIRST_GREETING=REDHATTERS -p SECOND_GREETING=WORLD --showlog
~~~

## 6. Start the pipeline using tkn CLI

~~~sh
tkn pipeline start greetings -p FIRST_GREETING=REDHATTERS -p SECOND_GREETING=WORLD --showlog
~~~

## 7. Create EventListener, TriggerBinding and TriggerTemplate resources in the namespace:

~~~sh
$ cd tekton-lab
$ oc create -k triggers
~~~

## 8. Expose the EventListener service:

~~~sh
oc expose svc/el-greetings-el
~~~

## 9. Start the Pipeline triggering an Event:

~~~sh
$ cd tekton-lab/test
$ curl -X POST -H "Content-Type: application/json" --data @body.json  http://el-greetings-el-devops.apps.cluster-5c71.5c71.sandbox1496.opentlc.com

$ tkn pr logs
~~~
