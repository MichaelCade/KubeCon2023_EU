# Data Consistency (Lab 4 )

In the previous example we performed a simple, crash consistent snapshot of a PVC, but think about sensitive workloads like MySQL or MongoDB that require data consistency to ensure data can be reliably restored. [***Kanister***](https://docs.kanister.io/) is an open-source framework that allows us to orchestrate data consistent backups at the application level.

With Kanister, you have a Kubernetes-native tool that can quiesce data services prior to creating a VolumeSnapshot *or* perform a logical backup of a data service independent of underlying infrastructure.

*Why perform a logical backup?* A key reason is that some data services, such as ***etcd***, simply don't provide the necessary primitives to lock and flush in order to capture a data consistent snapshot.

In this lab you will deploy and use the ***Kanister*** framework to implement a logical backup of your Pac-Man database.

📖 1. Install Pac-Man
=====================

Similar to Lab 2, the manifests for the Pac-Man application have been provisioned for you in `/lab4/` - *I promise this version works!*

1. Run the following to install the working version of Pac-Man:

    ```bash
    kubectl create namespace lab4
    cd /
    kubectl apply -f /lab4/ -n lab4
    ```

2. Run the following to monitor Pod status:

    ```bash
    watch kubectl get pods -n lab4
    ```

3. Once ***STATUS*** has reached ***Running*** for all Pods, press `CTRL+C` to end the `watch` command.

4. Open the ***pacman-svc NodePort*** tab in Instruqt and complete a game of Pac-Man, saving your name/initials to be written to the high score database as shown below:

    ![pacman highscore screenshot](/assets/pacman-highscore.png)

    > ℹ️ ***NOTE***
    >
    > Your high score may not appear in the list immediately. If so, click the ***Back*** button within Pac-Man and then select ***High Score*** from the bottom menu to confirm your score is visible.

📖 2. Install MinIO
===================

***MinIO*** is an open-source S3 compatible object storage solution which will be used for storing the backup data. It's also a great example of how Kubernetes can be used to transform block storage into other software defined storage services.

1. Run the following to install MinIO:

    ```bash
    helm repo add minio https://helm.min.io/ --insecure-skip-tls-verify
    kubectl create ns minio

    # Deploy minio with a pre-created "kanister-bucket" bucket, and "minioaccess"/"miniosecret" creds
    helm install minio minio/minio --namespace=minio --version 8.0.10 \
      --set persistence.size=5Gi \
      --set defaultBucket.enabled=true \
      --set defaultBucket.name=kanister-bucket \
      --set accessKey=minioaccess \
      --set secretKey=miniosecret
    ```

    > 🚩 ***WARNING***
    >
    > This S3 bucket will be running on the same primary storage as the application you're protecting - while this works for a lab environment, obviously you would never want to do this in the real world!

2. Run the following to monitor Pod status:

    ```bash
    watch kubectl get pods -n minio
    ```

3. Once ***STATUS*** has reached ***Running*** for all Pods, press `CTRL+C` to end the `watch` command.

4. Run the following command to apply a NodePort configuration to the cluster to expose the `minio` UI on port `30002`:

    ```yaml
    cat<<'EOF'> minio-nodeport.yaml | kubectl apply -f minio-nodeport.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: minio-nodeport
      namespace: minio
    spec:
      selector:
        app: minio
        release: minio
      ports:
      - name: http
        port: 9000
        nodePort: 30002
      type: NodePort
    EOF
    ```

📖 3. Install Kanister
======================

One of the core strengths of Kubernetes is its extensibility, the ability to add new APIs and capabilities through ***Custom Resource Definitions (CRDs)***. Kanister implements 3 Custom Resources on the cluster:

| **CRD**              | **Description**                                                                                                                                                                                             |
|------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ***Profiles***   | Define the connection and credential info for *where* backup data will be stored (e.g. S3 or NFS)                                                                                                       |
| ***Blueprints*** | Define the logic of *how* backup and restore operations will be performed                                                                                                                               |
| ***ActionSets*** | Used to initialize a specific ***Blueprint*** action (e.g. backup, restore), against a specific ***Deployment*** or ***StatefulSet***, using a specific ***Profile*** to upload or download backup data |

Kanister acts as a controller, monitoring for and executing new ***ActionSet*** resources on the cluster.

1. Install the Kanister controller on your cluster using the Kanister Helm chart:

    ```bash
    kubectl create namespace kanister
    helm repo add kanister https://charts.kanister.io/
    helm install kanister-operator --namespace kanister kanister/kanister-operator
    ```

2. Download the latest version of Kanister tools, `kanctl` and `kando` into your local Instruqt environment:

    ```
    curl https://raw.githubusercontent.com/kanisterio/kanister/master/scripts/get.sh | bash
    ```

    | **Executable** | **Description** |
    |---|---|
    | `kanctl` | Although all Kanister Custom Resources can be managed using kubectl, `kanctl` provides a simpler, dedicated interface to create ***Profiles*** and ***ActionSet*** resources. |
    | `kando` | A common use case for Kanister is to transfer data between Kubernetes and an object store like AWS S3. `kando` makes it simple to push and pull data without the cumbersome experience of passing ***Profile*** details to tools like `awscli` from inside a Blueprint. |


1. Apply the MongoDB StatefulSet Blueprint, which will use to protect Pac-Man MongoDB, to the cluster:

    ```yaml
    cat<<'EOF'> mongodb-blueprint.yaml | kubectl apply -f mongodb-blueprint.yaml -n kanister
    apiVersion: cr.kanister.io/v1alpha1
    kind: Blueprint
    metadata:
      name: mongodb-blueprint
    actions:
      # Logical Blueprints typically provide actions to backup, restore, and delete a backup from the repository
      backup:
        # Output Artifacts allows you to save information related to your ActionSet
        # In this case, we are persisting the timestamp generated path to your backup data as "backupLocation.path"
        outputArtifacts:
          backupLocation:
            keyValue:
              path: '/mongodb-replicaset-backups/{{ .StatefulSet.Name }}/{{ toDate "2006-01-02T15:04:05.999999999Z07:00" .Time  | date "2006-01-02T15-04-05" }}/rs_backup.gz'
        phases:
        # KubeTask provisions a new, temporary worker Pod
        - func: KubeTask
          name: takeConsistentBackup
          objects:
            # Objects allows you to pull in manifest data to use in Blueprint commands
            # Here this is used to get the Secret used for MongoDB auth
            mongosecret:
              kind: Secret
              name: '{{ .StatefulSet.Name }}'
              namespace: "{{ .StatefulSet.Namespace }}"
          args:
            # Provisions worker Pod in the same namespace as MongoDB
            namespace: "{{ .StatefulSet.Namespace }}"
            # Container image for worker Pod with tools to perform MongoDB backup and 'kando' push
            image: ghcr.io/kanisterio/mongodb:0.90.0
            # Command for worker Pod to run
            # Defines host string for connecting to MongoDB
            # Defines MongoDB password from mongosecret referenced in objects above
            # Command to perform data consistent dump database contents
            # Push backup data to location defined in ActionSet Profile
            # The path value will be saved as the "backupLocation" artifact
            command:
              - bash
              - -o
              - errexit
              - -o
              - pipefail
              - -c
              - |
                host='{{ .StatefulSet.Name }}-0.{{ .StatefulSet.Name }}.{{ .StatefulSet.Namespace }}.svc.cluster.local'
                dbPassword='{{ index .Phases.takeConsistentBackup.Secrets.mongosecret.Data "mongodb-root-password" | toString }}'
                dump_cmd="mongodump --gzip --archive --host ${host} -u root -p ${dbPassword}"
                ${dump_cmd} | kando location push --profile '{{ toJson .Profile }}' --path '/mongodb-replicaset-backups/{{ .StatefulSet.Name }}/{{ toDate "2006-01-02T15:04:05.999999999Z07:00" .Time  | date "2006-01-02T15-04-05" }}/rs_backup.gz' -
      restore:
        # We reference the backup location that was saved as an Output Artifact during Backup
        inputArtifactNames:
          - backupLocation
        phases:
        # Again, KubeTask will provision a new worker Pod
        - func: KubeTask
          name: restoreFromObjectStore
          objects:
            # Again, we reference the MongoDB secret for auth to perform restore
            mongosecret:
              kind: Secret
              name: '{{ .StatefulSet.Name }}'
              namespace: "{{ .StatefulSet.Namespace }}"
          args:
            namespace: "{{ .StatefulSet.Namespace }}"
            # We use the same container image for restore as it has our MongoDB tools and 'kando'
            image: ghcr.io/kanisterio/mongodb:0.90.0
            # Instead of a 'mongodump', we now want to perform a 'mongorestore' to repopulate the database
            # References the Input Artifact, which is where our backup data is within our S3 repository
            # Backup data is first downloaded with 'kando' and then piped into the 'mongorestore' command
            command:
              - bash
              - -o
              - errexit
              - -o
              - pipefail
              - -c
              - |
                host='{{ .StatefulSet.Name }}-0.{{ .StatefulSet.Name }}.{{ .StatefulSet.Namespace }}.svc.cluster.local'
                dbPassword='{{ index .Phases.restoreFromObjectStore.Secrets.mongosecret.Data "mongodb-root-password" | toString }}'
                restore_cmd="mongorestore --gzip --archive --drop --host ${host} -u root -p ${dbPassword}"
                kando location pull --profile '{{ toJson .Profile }}' --path '{{ .ArtifactsIn.backupLocation.KeyValue.path }}' - | ${restore_cmd}
      delete:
        inputArtifactNames:
          - backupLocation
        phases:
        - func: KubeTask
          name: deleteFromObjectStore
          args:
            namespace: "{{ .Namespace.Name }}"
            image: ghcr.io/kanisterio/mongodb:0.90.0
            # Deletes the backup from the path referenced in "backupLocation"
            command:
              - bash
              - -o
              - errexit
              - -o
              - pipefail
              - -c
              - |
                s3_path="{{ .ArtifactsIn.backupLocation.KeyValue.path }}"
                kando location delete --profile '{{ toJson .Profile }}' --path ${s3_path}
    EOF
    ```

    > ℹ️ ***NOTE***
    >
    > While this Blueprint only leverages the ***KubeTask*** function of Kanister to dynamically provision Pods to perform data protection operations, Kanister provides several different functions including taking snapshots, scaling workloads, and executing commands against existing Pods.
    >
    > You can review all of the available capabilities [in the Kanister documentation](https://docs.kanister.io/functions.html).

📖 4. Perform Logical Backup
============================

Before we can create the ActionSet to perform our Blueprints `backup` Action, we first need to define a Profile that Kanister will use to push the dump of MongoDB.

1. Create a Kanister ***Profile*** using the details from your MinIO installation:

    ```bash
    kanctl create profile \
      --namespace kanister \
      --bucket kanister-bucket \
      --skip-SSL-verification \
      --endpoint http://k8s-cluster.$INSTRUQT_PARTICIPANT_ID.instruqt.io:30002 \
      s3compliant \
      --access-key minioaccess \
      --secret-key miniosecret
    ```

2. View the ***Profile*** object created by `kanctl`:

    ```bash
    kubectl get profiles -n kanister -o yaml
    ```

3. Generate a Kanister ***ActionSet*** to execute the `backup` action:

    ```bash
    # Return the generated name of the Profile created in Step 1
    KANISTER_PROFILE=$(kubectl get profiles.cr.kanister.io -n kanister -ojsonpath='{.items[0].metadata.name}')

    # --action backup - Which action in the Blueprint to run
    # --blueprint mongodb-blueprint - Which Blueprint to use
    # --statefulset lab4/pacman-mongodb - Which workload to apply the Blueprint action
    # --profile kanister/$KANISTER_PROFILE - Which Profile for 'kando' to use
    kanctl create actionset \
      --action backup \
      --namespace kanister \
      --blueprint mongodb-blueprint \
      --statefulset lab4/pacman-mongodb \
      --profile kanister/$KANISTER_PROFILE
    ```

4. Check the status of the backup ***ActionSet***:

    ```bash
    BACKUP_ACTIONSET=$(kubectl get actionset -n kanister --sort-by=.metadata.creationTimestamp| grep backup |awk '{print $1}'|tail -1)
    kubectl get actionset -n kanister $BACKUP_ACTIONSET -ojsonpath='{.status}' |jq
    ```

    If `state` is still `running`, wait a few seconds and re-run the command. In addition to the ***ActionSet*** completion status, you should observe the path to your backup data:

    ```json,nocopy
    {
      "actions": [
        {
          "artifacts": {
            "backupLocation": {
              "keyValue": {
                # Path to database dump
                "path": "/mongodb-replicaset-backups/pacman-mongodb/2023-04-09T16-40-32/rs_backup.gz"
              }
            }
          },
          "blueprint": "mongodb-blueprint",
          "deferPhase": {
            "name": "",
            "state": ""
          },
          "name": "backup",
          "object": {
            "apiVersion": "",
            "group": "",
            "kind": "statefulset",
            "name": "pacman-mongodb",
            "namespace": "lab4",
            "resource": ""
          },
          "phases": [
            {
              "name": "takeConsistentBackup",
              "state": "complete"
            }
          ]
        }
      ],
      "error": {
        "message": ""
      },
      "progress": {
        "lastTransitionTime": "2023-04-09T16:40:38Z",
        "percentCompleted": "10.00"
      },
      "state": "complete"
    }
    ```

5. From the ***MinIO NodePort*** tab in Instruqt, validate you can view the MongoDB dump file available at the path from the previous step.

    - ***Access Key*** - minioaccess
    - ***Secret Key*** - miniosecret

    ![minio UI](/assets/rs-backup-gz.png)

📖 5. Restore Your Backup
=========================

1. From the ***Terminal***, drop the `pacman` database from MongoDB that holds your highscore:

    ```bash
    export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace lab4 pacman-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)
    kubectl exec -it pacman-mongodb-0 -n lab4 -- mongosh pacman --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD --eval 'db.dropDatabase();'
    ```

2. Return to the ***pacman-svc NodePort*** and validate your high scores are gone!

    ![no high scores](/assets/no-highscores.png)

3. Restore your database from your most recent Kanister backup:

    ```bash
    # Get the name of your most recent backup
    BACKUP_ACTIONSET=$(kubectl get actionset -n kanister --sort-by=.metadata.creationTimestamp| grep ^backup |awk '{print $1}'|tail -1)
    kanctl --namespace kanister create actionset --action restore --from $BACKUP_ACTIONSET
    ```

4. Check the status of the restore ***ActionSet***:

    ```bash
    RESTORE_ACTIONSET=$(kubectl get actionset -n kanister --sort-by=.metadata.creationTimestamp | grep restore |awk '{print $1}'|tail -1)
    watch kubectl get actionset -n kanister $RESTORE_ACTIONSET
    ```

5. Once the ***STATE*** of the previous command changes to ***Complete***, return to Pac-Man and validate your high score is back🥳.

🏁 Part 6. Takeaways
====================

Many stateful data services on Kubernetes require more than simple crash-consistent snapshots to ensure data can be reliably recovered.

Kanister is built for Kubernetes to enable you perform data consistent backups at the application level, whether integrated with VolumeSnapshots, or as logical backups.
