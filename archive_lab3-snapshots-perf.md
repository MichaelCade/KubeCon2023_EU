# VolumeSnapshots & Performance (Bonus - Lab 3)

All YAML manifests although you will find full commands below, we have included them in the Lab1 folder withinn the repository. 

- Volume Snapshots 
- Kubestr 
- Install Kanister 
- Object Storage 
- Kanister Blueprints 
- Backup
- Restore

# Volume Snapshots

In this exercise we are going to create a volume snapshot.

When you create the volume snapshot, you have to register Volume Snapshot Classes to your cluster. The default VolumeSnapshotClass called `csi-hostpath-snapclass`. You can confirm by running the following command:

```bash
kubectl get volumesnapshotclasses
```

create namespace 

`kubectl create ns task5 `

Next, create a persistent volume claim to create a persistent volume dynamically:

```bash
cat <<'EOF'> csi-pvc.yaml | kubectl apply -f csi-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
  namespace: task5
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: csi-hostpath-sc
EOF
```

You can confirm that persistent volume is created by the following command:

```bash
kubectl get pv | grep task5
```

# Take a volume snapshot

You can take a volume snapshot for the persistent volume claim:

```bash
cat <<'EOF'> snapshot-demo.yaml | kubectl apply -f snapshot-demo.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: snapshot-demo
  namespace: task5
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: csi-pvc
EOF
```

The key terms in the above snippet are listed below:

- `persistentVolumeClaimName` is the name of the PersistentVolumeClaim data source for the snapshot.

    A volume snapshot can request a particular class by specifying the name of a VolumeSnapshotClass using the attribute `volumeSnapshotClassName`. If nothing is set, then the default class is used.

You can confirm your volume snapshot by the following command:

```bash
kubectl get volumesnapshot -n task5
```

# Restore from a volume snapshot

In order to perform a restore operation, we are going to source the snapshot we created in a new persistent volume claim resource:

```bash
cat <<'EOF'> csi-pvc-restore.yaml | kubectl apply -f csi-pvc-restore.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc-restore
  namespace: task5
spec:
  storageClassName: csi-hostpath-sc
  dataSource:
    name: snapshot-demo
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

The key terms in the above snippet are listed below:

- `dataSource`: It specifies the snapshot resources to restore the data from.

You can confirm that a persistent volume claim was created from the `VolumeSnapshot`:

```bash
kubectl get pvc -n task5
```
# Kubestr

Kubestr is a collection of tools to discover, validate and evaluate your kubernetes storage options.

As adoption of Kubernetes grows so have the persistent storage offerings that are available to users. The introduction of [CSI](https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/) (Container Storage Interface) has enabled storage providers to develop drivers with ease. In fact there are around a 100 different CSI drivers available today. Along with the existing in-tree providers, these options can make choosing the right storage difficult.

Kubestr can assist in the following ways:

- Identify the various storage options present in a cluster.
- Validate if the storage options are configured correctly.
- Evaluate the storage using common benchmarking tools like FIO.

# Using Kubestr

## To install the tool

- Download the latest release:
I am using WSL so I am grabbing the latest release for Linux but other OS options are available - https://github.com/kastenhq/kubestr/releases

```
curl -sLO https://github.com/kastenhq/kubestr/releases/download/v0.4.37/kubestr_0.4.37_Linux_amd64.tar.gz
```

- Unpack the tool and make it an executable chmod +x kubestr.

```
sudo tar -xvzf kubestr_0.4.37_Linux_amd64.tar.gz -C /usr/local/bin
sudo chmod +x /usr/local/bin/kubestr
```

To discover available storage options: Run `kubestr`

## To run an FIO test

Using `kubestr` we can test sequential read and write speeds which are expected to be high.

Random reads and writes can be tested as well. Moreover, random writes separates good drives from the bad ones.

- Copy the test specification:

```
cat <<'EOF'>ssd-test.fio
[global]
bs=4k
ioengine=libaio
iodepth=1
size=1g
direct=1
runtime=10
directory=/
filename=ssd.test.file

[seq-read]
rw=read
stonewall

[rand-read]
rw=randread
stonewall

[seq-write]
rw=write
stonewall

[rand-write]
rw=randwrite
stonewall
EOF
```

- Run the following command to start the test
```
kubestr fio -f ssd-test.fio -s csi-hostpath-sc
```

- Additional options like --size and --fiofile can be specified.


## Installing Kanister

Pre-reqs are that we will also require helm here. 

- Let's create a namespace for our Kanister environment:

  ```bash
  kubectl create namespace kanister
  ```

- Add Kanister charts

  ```bash
  helm repo add kanister https://charts.kanister.io/
  ```

- Install the Kanister operator controller using helm

  ```bash
  helm install kanister-operator --namespace kanister kanister/kanister-operator
  ```

There are two command-line tools that are built within the Kanister repository.

## Kanctl

Although all Kanister custom resources can be managed using kubectl, there are situations where this may be cumbersome. A canonical example of this is backup/restore.

`kanctl` simplifies this process by allowing the user to create custom Kanister resources such as ActionSets and Profiles.

## Kando

A common use case for Kanister is to transfer data between Kubernetes and an object store like AWS S3.

We've found it can be cumbersome to pass Profile configuration to tools like the AWS command line from inside Blueprints.

`kando`  is a tool to simplify object store interactions from within blueprints. It also provides a way to create desired output from a blueprint phase.

## Install the tools

The script installs both `kanctl` and `kando`

```bash
curl https://raw.githubusercontent.com/kanisterio/kanister/master/scripts/get.sh | bash
```

Verify the installation:

```bash
kanctl --version
kando --version
```

# Object Storage 

we are going to configure a local S3-compatible object storage system using [MinIO](https://min.io/) in order to use as storage for backup data.

## This is not recommended as we will be placing our backups onto our "production" cluster, not a great place to be if you are trying to recover from a cluster wide failure! 

[MinIO](https://min.io/) is a High Performance Object Storage released under Apache License v2.0. It is API compatible with Amazon S3 cloud storage service.

```
helm repo add minio https://helm.min.io/ --insecure-skip-tls-verify
kubectl create ns minio
helm install kanister-minio minio/minio --namespace=minio --version 8.0.10 \
  --set persistence.size=20Gi \
  --set defaultBucket.enabled=true \
  --set defaultBucket.name=kanister-bucket \
  --set accessKey=minioadmin \
  --set secretKey=minioadmin \
  --wait
```

To access Minio from localhost, run the below commands:

```
export POD_NAME=$(kubectl get pods --namespace minio -l "release=kanister-minio" -o jsonpath="{.items[0].metadata.name}")
```

Open your browser to http://127.0.0.1:9090 and login with the token from the above step.

Use the following credentials to authenticate: 

accessKey=minioadmin
secretKey=minioadmin

# Deploy Pac-Man 
Should be deployed if we have followed along with Le steps and should now be fully functional 

# Kanister Blueprints 
We will need to make sure we have a blueprint that can protect our mongodb 

# Backup 
Walkthrough creating a kanister backup with the blueprint to object storage. 

# Failure Scenario 
Create a simple exec and connect to cause a problem with our data, or we could just invoke a bad score 

# Restore 
Restore to before the bad score was added. 