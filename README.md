# KubeCon2023_EU

This repository is to be used as a walkthrough location for our session at KubeCon 2023 in Amsterdam. 

## What went wrong with my persistent data?

Looking for the fundamentals around Kubernetes storage with volumes, especially persistent volumes, persistent volume claims, and storage classes? Then this tutorial is for you. Michael and Le will introduce some fundamental concepts, then dive right into some common pitfalls of setting up persistent data storage in Kubernetes clusters in the format of troubleshooting labs. During these hands-ons labs, folks will be presented with some common errors when setting up pods with persistent volumes or persistent volume claims in a cluster, and will work to identify and resolve these issues. They will be provided with tips and tools on how to navigate these scenarios. Newcomers to K8S will leave with an understanding of basic concepts and tools to effectively work with and troubleshoot persistent data. Additionally, they will also walk away with suggestions around data management for data services, and how to protect their data.

This session will be a 90-minute workshop 

More details can be found [here.](https://kccnceu2023.sched.com/event/1Hyah/tutorial-what-went-wrong-with-my-persistent-data-le-tran-michael-cade-kasten-by-veeam?iframe=no)

To follow along we will have 4 labs available all of which can be ran using minikube. 

### Install Minikube 

- [Installing Minikube](https://minikube.sigs.k8s.io/docs/start/)

The minikube installation should also install kubectl or the Kubernetes CLI, you will need this, again available through most package managers cross platform (Chocolatey, apt etc.)

We will also need helm to deploy some of our data services. 

- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [helm](https://helm.sh/docs/intro/install/)

