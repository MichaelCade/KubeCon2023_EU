kubectl create ns demo

cat postgres-no-pv.yaml

kubectl apply -f postgres-no-pv.yaml

watch kubectl get pods -n demo

kubectl -n demo exec -it postgres-0 bash

psql --username=admin postgresdb

CREATE TABLE KUBECON2023(
   ID INT PRIMARY KEY     NOT NULL,
   NAME           TEXT    NOT NULL,
   AGE            INT     NOT NULL,
   ADDRESS        CHAR(50),
   SALARY         REAL
);

\dt 

\q

exit 

kubectl delete pod postgres-0 -n demo

kubectl get pods -n demo 

kubectl -n demo exec -it postgres-0 bash

psql --username=admin postgresdb

\dt 

\q 

exit 

kubectl get pvc -n demo 