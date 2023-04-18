# StatefulSets - Task 3 

This section will allow us to spin up a stateful set.

Deployments and StatefulSets are both controllers in Kubernetes used for managing containerized applications, but they have different use cases. Deployments are used for managing stateless applications, where each instance of the application can be identical and can be terminated and replaced at any time. Deployments do not guarantee stable network identities or persistent storage. StatefulSets, on the other hand, are used for managing stateful applications that require stable network identities and persistent storage, such as databases or other distributed systems. StatefulSets ensure that each pod is created and terminated in a predictable, consistent order, and maintains a stable hostname for each pod. This is particularly important for stateful applications that rely on stable network identities. In summary, Deployments are ideal for stateless applications, while StatefulSets are better suited for stateful applications with strict ordering and consistency requirements.

`kubectl create ns task3`

Secret 
```
cat<<'EOF'> mongo-secret.yaml | kubectl apply -f mongo-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-users-secret
  namespace: task3
type: Opaque 
data:
  database-admin-name: Y2x5ZGU=
  database-admin-password: Y2x5ZGU=
  database-name: cGFjbWFu
  database-password: cGlua3k=
  database-user: Ymxpbmt5
EOF
```

Persistent Volume Claim 
```
cat<<'EOF'> mongo-pvc.yaml | kubectl apply -f mongo-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mongo-storage
  namespace: task3
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

StatefulSet
```
cat<<'EOF'> mongo-sfs.yaml | kubectl apply -f mongo-sfs.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    name: mongo
  name: mongo
  namespace: task3
  annotations:
spec:
  replicas: 1
  serviceName: mongo
  selector:
    matchLabels:
      name: mongo
  template:
    metadata:
      labels:
        name: mongo
    spec:
      initContainers:
      - args:
        - |
          mkdir -p /bitnami/mongodb
          chown -R "1001:1001" "/bitnami/mongodb"
        command:
        - /bin/bash
        - -ec
        image: docker.io/bitnami/bitnami-shell:10-debian-10-r158
        imagePullPolicy: Always
        name: volume-permissions
        resources: {}
        securityContext:
          runAsUser: 0
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /bitnami/mongodb
          name: mongo-db
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 1001
      serviceAccountName: default
      terminationGracePeriodSeconds: 30
      volumes:
      - name: mongo-db
        persistentVolumeClaim:
          claimName: mongo-storage
      containers:
      - image: bitnami/mongodb:4.4.8
        name: mongo
        env:
        - name: MONGODB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              key: database-admin-password
              name: mongodb-users-secret
        - name: MONGODB_DATABASE
          valueFrom:
            secretKeyRef:
              key: database-name
              name: mongodb-users-secret
        - name: MONGODB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: database-password
              name: mongodb-users-secret
        - name: MONGODB_USERNAME
          valueFrom:
            secretKeyRef:
              key: database-user
              name: mongodb-users-secret
        readinessProbe:
          exec:
           command:
            - /bin/sh
            - -i
            - -c
            - mongo 127.0.0.1:27017/$MONGODB_DATABASE -u $MONGODB_USERNAME -p $MONGODB_PASSWORD
              --eval="quit()"
        ports:
        - name: mongo
          containerPort: 27017
        volumeMounts:
          - name: mongo-db
            mountPath: /bitnami/mongodb/
EOF
```

Service
```
cat<<'EOF'> mongo-svc.yaml | kubectl apply -f mongo-svc.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    name: mongo
  name: mongo
  namespace: task3
spec:
  type: ClusterIP
  ports:
    - port: 27017
      targetPort: 27017
  selector:
    name: mongo
EOF
```

`watch kubectl get pods -n task3` 

When running, you have a statefulset, you will notice that the naming is different to the naming convention used in deployments, ordinal numbers are used and a graceful removal is used when scaled down. 

At this stage we will move back to slides or move into Lab 2. 