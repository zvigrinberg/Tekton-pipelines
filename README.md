# Tekton-pipelines

- This repo will contain both tekton tasks and pipelines

- This repo will contain also Triggers and EventListeners. 

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
