# VolumeSnapshot (Lab 3)

In this lab you will work with ***VolumeSnapshots***, creating and restoring snapshots of your Kubernetes Persistent Volumes. VolumeSnapshots are an ***optional*** capability, defined as part of the ***Container Storage Interface (CSI)*** specification, that can be implemented by the storage provider.

📖 1. Create A Volume Snapshot
==============================

Similar to how ***StorageClasses*** define available classes of storage and their configuration, ***VolumeSnapshotClasses*** define a CSI-based storage provider and behavior, primarily whether or not to delete underlying storage snapshots when VolumeSnapshots are removed from the cluster.

1. In the ***Terminal***, run the following to view the list of available VolumeSnapshotClasses on your cluster:

    ```
    kubectl get volumesnapshotclasses
    ```

    You should observe only a single option, `csi-hostpath-sc`.

    > ℹ️ ***NOTE***
    >
    > This is why the lab provides the Hostpath CSI driver in addition to the built-in hostpath function of Kubernetes. Despite still only providing non-production, local storage, the Hostpath CSI implements the CSI Volume Snapshot specification, whereas the built-in option has no snapshot capability.

2. Run the following to provision a test Pod, `snap-pod`, which mounts a dynamically provisioned 1GiB volume to store the file `/data/index.html`, using the Hostpath CSI storage provisioner:

    ```yaml
    kubectl create namespace lab3
    cat<<'EOF'> snap-pod.yaml | kubectl apply -f snap-pod.yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: csi-pvc
      namespace: lab3
    spec:
      storageClassName: csi-hostpath-sc
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: snap-pod
      namespace: lab3
    spec:
      volumes:
      - name:  myvol
        persistentVolumeClaim:
          claimName: csi-pvc
      containers:
      - image: busybox
        imagePullPolicy: IfNotPresent
        name: busybox
        tty: true
        volumeMounts:
        - name: myvol
          mountPath: /data
    EOF
    ```

3. Run the following to monitor Pod status:

    ```bash
    watch kubectl get pod snap-pod -n lab3
    ```

4. Once ***STATUS*** has reached ***Running***, press `CTRL+C` to end the `watch` command.

5. Write a file to `csi-pvc` and verify its contents can be read:

    ```
    kubectl exec snap-pod -n lab3 -- /bin/sh -c 'echo "Gosh, I sure hope no one deletes me!" > /data/index.html'
    kubectl exec snap-pod -n lab3 -- cat /data/index.html
    ```

6. Run the following to create a VolumeSnapshot, `snapshot-demo`, of the PVC being used by `snap-pod`:

    ```yaml
    cat <<'EOF'> snapshot-demo.yaml | kubectl apply -f snapshot-demo.yaml
    apiVersion: snapshot.storage.k8s.io/v1
    kind: VolumeSnapshot
    metadata:
      name: snapshot-demo
      # VolumeSnapshots exist in the same namespace as the PVCs they reference
      namespace: lab3
    spec:
      # Specifies which SnapshotClass should be used
      volumeSnapshotClassName: csi-hostpath-snapclass
      source:
        # Specifies which PVC to snapshot
        persistentVolumeClaimName: csi-pvc
    EOF
    ```

7. While most snapshot operations only take a matter of seconds, you can confirm the VolumeSnapshot is complete by checking the ***Ready to use*** status:

    ```
    kubectl get volumesnapshot -n lab3
    ```

📖 2. Restore from a Volume Snapshot
====================================

If you accidentally delete some of my persistent data, or introduce some incorrect or malicious data, VolumeSnapshots can provide a means of quickly restoring your data to a different point in time.

1. In the ***Terminal***, run the following to delete your precious `/data/index.html` file and verify it is no longer available:

    ```
    kubectl exec snap-pod -n lab3 -- rm -f /data/index.html
    kubectl exec snap-pod -n lab3 -- ls -l /data/
    ```

    There should be 0 files in `/data/`.

1. Delete your existing Pod and `csi-pvc` PVC, which will also remove the associated PV and any underlying data on our physical storage:

    ```
    kubectl delete -f snap-pod.yaml -n lab3
    ```

    > ℹ️ ***NOTE***
    >
    > Kubernetes will not "finalize" the deletion of any PVC that is still actively mounted by a Pod, this is why you are deleting the Pod in addition to the PVC. In a more typical environment, a replica Pod within a Deployment or StatefulSet would be scaled down to allow the PVC to be restored, and then scaled back up.

1. Apply the following to re-create `snap-pod` as well as a PVC referencing your VolumeSnapshot:

    ```yaml
    cat<<'EOF'> snap-pod-restore.yaml | kubectl apply -f snap-pod-restore.yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: csi-pvc
      namespace: lab3
    spec:
      #### Specifies which VolumeSnapshot to use as the source of the PVC
      dataSource:
        name: snapshot-demo
        kind: VolumeSnapshot
        apiGroup: snapshot.storage.k8s.io
      ###
      storageClassName: csi-hostpath-sc
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: snap-pod
      namespace: lab3
    spec:
      volumes:
      - name:  myvol
        persistentVolumeClaim:
          claimName: csi-pvc
      containers:
      - image: busybox
        imagePullPolicy: IfNotPresent
        name: busybox
        tty: true
        volumeMounts:
        - name: myvol
          mountPath: /data
    EOF
    ```

    The only difference between the manifest above and the original manifest is the inclusion of the `dataSource` in the PVC spec. This field allows you to provision what is effectively a clone based on the specified VolumeSnapshot.

1. Run the following to monitor Pod status:

    ```bash
    watch kubectl get pod snap-pod -n lab3
    ```

1. Once ***STATUS*** has reached ***Running***, press `CTRL+C` to end the `watch` command.

1. Validate you've got that sweet, sweet data back:

    ```
    kubectl exec snap-pod -n lab3 -- cat /data/index.html
    ```

🏁 Part 3. Takeaways
====================

- The CSI VolumeSnapshot standard makes it possible to programmatically perform snapshot and restore operations for persistent data across a wide range of storage solutions that can be used with Kubernetes
- Not every CSI provider supports VolumeSnapshots, [best to do your research and validate!](https://kubernetes-csi.github.io/docs/drivers.html)

    >💡 ***TIP***
    >
    > [Kubestr](https://kubestr.io/) is an open source tool that can be used to quickly validate the capabilities across available storage classes on your cluster, validate configurations, and evaluate storage performance using FIO.
