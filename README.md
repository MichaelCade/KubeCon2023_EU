# KubeCon2023_EU

This repository is to be used as a walkthrough location for our session at KubeCon 2023 in Amsterdam. 

## What went wrong with my persistent data?

Looking for the fundamentals around Kubernetes storage with volumes, especially persistent volumes, persistent volume claims, and storage classes? Then this tutorial is for you. Michael and Le will introduce some fundamental concepts, then dive right into some common pitfalls of setting up persistent data storage in Kubernetes clusters in the format of troubleshooting labs. During these hands-ons labs, folks will be presented with some common errors when setting up pods with persistent volumes or persistent volume claims in a cluster, and will work to identify and resolve these issues. They will be provided with tips and tools on how to navigate these scenarios. Newcomers to K8S will leave with an understanding of basic concepts and tools to effectively work with and troubleshoot persistent data. Additionally, they will also walk away with suggestions around data management for data services, and how to protect their data.

This session will be a 90-minute workshop 

More details can be found [here.](https://kccnceu2023.sched.com/event/1Hyah/tutorial-what-went-wrong-with-my-persistent-data-le-tran-michael-cade-kasten-by-veeam?iframe=no)

To follow along we will have 2 labs available all of which can be ran using minikube. 

# Pre-Reqs
- Install Minikube (Shown Below)
# [Kubernetes Storage (Lab 1)](/lab1-storage.md)
- Ephemeral Volumes 
- Projected Volumes 
- Persistent Volumes 
    - Persistent Volume Claims
- Storage Classes 
    - Persistent Volumes
    - Persistent Volume
# [Storage Troubleshooting (Lab 2)](/lab2-troubleshooting.md)
- Deploy Application 
- Troubleshoot
- Get your high scores 
# [VolumeSnapshots & Performance (Bonus - Lab 3)](/lab3-snapshots-perf.md)
- Volume Snapshots 
- Kubestr 
- Install Kanister 
- Object Storage 
- Kanister Blueprints 
- Backup
- Restore

## Getting Started Instructions

- Minikube installation 
- Minikube version check 
- Start Minikube cluster 

### Minikube installation 
First we need a local Kubernetes cluster to work on, I have chosen [minikube](https://minikube.sigs.k8s.io/docs/start/)

I am using WSL so I used the following command, other OS instructions can be found on the link above. 

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

You could also use [Arkade](https://github.com/alexellis/arkade) with the following command: 

`arkade get minikube`

### Minikube version check 
Now that we have minikube installed check the version with this command: 
`minikube update-check` 

The below is the output and is what we will be using for this lab workshop. 

```
➜ demo minikube update-check
CurrentVersion: v1.29.0
LatestVersion: v1.29.0
```

### Start Minikube cluster
We will be using [Docker](https://minikube.sigs.k8s.io/docs/drivers/docker/) as our Container or virtual machine manager for minikube you will see though that this could be a list of other options. such as: Docker, QEMU, Hyperkit, Hyper-V, KVM, Parallels, Podman, VirtualBox, or VMware Fusion/Workstation.

We will start our cluster with the following command: 

`minikube start --addons volumesnapshots,csi-hostpath-driver -p kubecon23`

