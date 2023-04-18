# Trouble shooting Kubernetes Cluster (Lab 2)
The goal of this lab is to deploy a Pacman application and play the game in your browser.
Starting screen| Game started
:---:|:---:
![picture](/assets/pacman-1.png) | ![picture](/assets/pacman-2.png)

Application overview
====================
The Pacman application consists of the following components:

Name| Type| Description
---|---|---
pacman-deployment.yaml|	Deployment|	Frontend Pac-Man application written in Node.js
mongodb-statefulset.yaml| StatefulSet| Backend database for storing high scores using persistent storage
mongodb-scripts-config-map.yaml| ConfigMap| Injects startup scripts into MongoDB container
mongodb-secret.yaml| Secret| Secret used to authenticate to MongoDB
service-acount.yaml| ServiceAccount| Used by MongoDB
mongodb-svc.yaml| ClusterIP Service| Used by Pac-Man to access MongoDB on port 27017
pacman-svc.yaml| NodePort Service| Used to expose Pac-Man frontend on host port 30001
static-pv.yaml| PersistentVolume| Provisions storage for MongoDB backend
static-pvc.yaml| PersistentVolumeClaim| Claims storage for MongoDB backend

Task overview
=============

Each of these manifests have been provisioned for you in `~/lab2/`.

To create the resources in this application, run the following commands:
```bash
cd Lab2
kubectl apply -f .
```

However, there is one or more problems with these manifests that prevent the whole application to be successfully deployed. Our task is to identify the problem(s) and provide solution(s).

Once the application is running, access the game with the following commands:
```bash
export POD_NAME=$(kubectl get pods --namespace lab2 -l "app.kubernetes.io/name=pacman,app.kubernetes.io/instance=pacman" -o jsonpath="{.items[0].metadata.name}")
export CONTAINER_PORT=$(kubectl get pod --namespace lab2 $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
kubectl --namespace lab2 port-forward $POD_NAME 8080:$CONTAINER_PORT
```

And then access the game at  http://127.0.0.1:8080


Useful Kubectl commands
=======================
Below is short list of `kubectl` commands to interact with resources in a cluster:

A more comprehensive list is available at  https://kubernetes.io/docs/reference/kubectl/cheatsheet/​

Purpose | Command
--- | ---
Set namespace for all subsequent commands | `kubectl config set-context –-current-context --namespace=<ns>​`
Overview of pods, deployment, services, and replicaset​ | `kubectl get all –n <ns>`
List pods in a namespace  | `kubectl get pods -n <ns>`
Get a specific pod | `kubectl get pod <my_pod> -n <ns> -<format>(owide, oyaml, ojson)`
View human-readable description of the object, including related objects and events​| `kubectl describe pod –n <ns>`
See the logs for a running container ​ | `kubectl logs pod –n <ns> -c <container_name> -f`
Create a new resource or modify an exisiting resource | `kubectl apply –f <manifest_file(s)> -n <ns>`
Edit an existing resource from the default editor | `kubectl edit <resource> <resource_name> -n <ns>`
Update field(s) of a resource | `kubectl patch <resource> <resource_name> -n <ns> -p '<field_to_be_updated>'`
Delete resource ​through name | `kubectl delete <resource> <resource_name> -n <ns>`
Delete resource through manifest file | `kubectl delete –f <manifest_file>`
Interactive shell access to a running pod​ | `kubectl exec <pod_name> -n <ns> --stdin --tty  -- /bin/sh`

​

