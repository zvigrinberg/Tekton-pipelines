# Tekton-pipelines

- This repo will contain both tekton tasks and pipelines

- This repo will contain also Triggers, TriggerBindings, TriggerTemplates and EventListeners. 

## Prerequisites:

You'll need an openshift or k8 cluster, if you don't have access to such, 
you can create a 1 node local kubernetes cluster using [minikube](https://minikube.sigs.k8s.io/docs/start/)
or kubernetes cluster in a container using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

## Install Tekton Pipelines

- To install Tekton after connecting to the cluster, run the following:

```shell
kubectl apply --filename \
https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

- See progression of installation
```shell
kubectl get pods -n tekton-pipelines -w
```

## Create a sample task and taskRun
```shell
cat > task-example.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: echo
      image: alpine
      script: |
        #!/bin/sh
        echo "Hello World"
EOF

kubectl apply -f task-example.yaml 
```

```shell
cat > taskRun-example.yaml << EOF
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: hello-task-run
spec:
  taskRef:
    name: hello
EOF
kubectl apply -f taskRun-example.yaml 
```

- See a status of the taskRun object:
```shell
kubectl get taskrun hello-task-run
```

- See logs of the taskRun
```shell
kubectl logs --selector=tekton.dev/taskRun=hello-task-run
```
## Create a sample pipeline and pipelineRun
- Create a goodbye task
```shell
cat > goodbye-world.yaml << EOF
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
   name: goodbye
   spec:
   params:
   - name: username
   type: string
   steps:
   - name: goodbye
   image: ubuntu
   script: |
   #!/bin/bash
   echo "Goodbye \$(params.username)!"
   EOF
```
- Apply it to the cluster
```shell
kubectl apply -f goodbye-world.yaml
```
- Create a pipeline resource, and apply it to the cluster
```shell
cat > hello-goodbye-pipeline.yaml << EOF
   apiVersion: tekton.dev/v1beta1
   kind: Pipeline
   metadata:
   name: hello-goodbye
   spec:
   params:
   - name: username
   type: string
   tasks:
   - name: hello
   taskRef:
   name: hello
   - name: goodbye
   runAfter:
   - hello
   taskRef:
   name: goodbye
   params:
   - name: username
   value: \$(params.username)
   EOF

   kubectl apply --filename hello-goodbye-pipeline.yaml
```

- Create a pipelineRun resource to run pipeline:
```shell
cat > hello-goodbye-pipeline-run.yaml << EOF
   apiVersion: tekton.dev/v1beta1
   kind: PipelineRun
   metadata:
   name: hello-goodbye-run
   spec:
   pipelineRef:
   name: hello-goodbye
   params:
   - name: username
   value: "Tekton"
   EOF
   
   kubectl apply --filename hello-goodbye-pipeline-run.yaml
```

- Follow logs of pipelineRun by entering the following command:
```shell
tkn pipelinerun logs hello-goodbye-run -f -n default
```
## Create a sample EventListener , TriggerTemplate and TriggerBinding
- First, if not installed, install Tekton Triggers:
```shell
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml

# Watch the progress of installation, until everything is finished.
kubectl get pods --namespace tekton-pipelines --watch
```
- Create A TriggerTemplate instance to connect Pipeline to the Trigger emitted by EventListener
```shell
cat > trigger-template.yaml << EOF
   apiVersion: triggers.tekton.dev/v1beta1
   kind: TriggerTemplate
   metadata:
   name: hello-template
   spec:
   params:
   - name: username
   default: "Kubernetes"
   resourcetemplates:
   - apiVersion: tekton.dev/v1beta1
   kind: PipelineRun
   metadata:
   generateName: hello-goodbye-run-
   spec:
   pipelineRef:
   name: hello-goodbye
   params:
   - name: username
   value: \$(tt.params.username)
   EOF
   kubectl apply -f trigger-template.yaml
   
```

- Create A TriggerBinding that will parse the parameters from the event payload and assign them to the TriggerTemplate's parameters:
```shell
cat > trigger-binding.yaml << EOF
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: hello-binding
spec: 
  params:
  - name: username
    value: \$(body.username)
EOF

kubectl apply -f trigger-template.yaml
```
- Create an EventListener that will Listen to request using webhooks , and will create events/triggers, and will connect them to the created TriggerTemplate and TriggerBinding resources just created:
```shell
kubectl apply -f trigger-binding.yaml
   cat > event-listener.yaml << EOF
   apiVersion: triggers.tekton.dev/v1beta1
   kind: EventListener
   metadata:
   name: hello-listener
   spec:
   serviceAccountName: tekton-robot
   triggers:
   - name: hello-trigger
   bindings:
   - ref: hello-binding
   template:
   ref: hello-template
   EOF
   
```
- Create RBAC permissions for all operations:
```shell
cat > rbac.yaml << EOF
   apiVersion: v1
   kind: ServiceAccount
   metadata:
   name: tekton-robot
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: RoleBinding
   metadata:
   name: triggers-example-eventlistener-binding
   subjects:
   - kind: ServiceAccount
   name: tekton-robot
   roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: tekton-triggers-eventlistener-roles
   ---
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
   name: triggers-example-eventlistener-clusterbinding
   subjects:
   - kind: ServiceAccount
   name: tekton-robot
   namespace: default
   roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: tekton-triggers-eventlistener-clusterroles
   EOF
   
    kubectl apply -f rbac.yaml
```
- Only In case Istio is enabled in the cluster, you need to disable automatic sidecar injection to default namespace:
```shell
kubectl label namespace default istio-injection=disabled --overwrite
```

- Create The EventListener in the cluster:
```shell
kubectl apply -f event-listener.yaml
```

- In a new window, Expose and forward port 8080 of service el-hello-listener to client' host
```shell
kubectl port-forward svc/el-hello-listener 8080
```

- Invoke the webhook endpoint(listened by EventListener' pod):
```shell
curl -v -H 'content-Type: application/json' -d '{"username": "Tekton"}' http://localhost:8080
```
- See logs of pipelines that was activated:
```shell
   kubectl get pipelineruns -w
   kubectl logs hello-goodbye-run-bb6gs
   kubectl describe pipelinerun hello-goodbye-run-bb6gs
   tkn pipelinerun logs hello-goodbye-run-bb6gs -f
```

