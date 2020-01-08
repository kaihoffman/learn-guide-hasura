# Hasura on Civo Cloud

## Introduction

Civo managed Kubernetes is currently in beta, and has a number of applications on the Kubernetes marketplace that can be easily installed on a cluster. At the time of me writing this guide, Hasura is not yet on the marketplace, so I wrote this guide to mount Hasura in the Civo k3s service. I hope you find it useful!

If you are not yet a member of the Civo Kubernetes #KUBE100 beta, you can [apply to join here](https://www.civo.com/kube100). All testers get a monthly allowance of $70 for the duration of the beta in exchange for their feedback.

## What is Hasura?

According to the official site:

> Hasura is an [open source](https://github.com/hasura/graphql-engine) engine that connects to your databases & microservices and instantly gives you a production-ready GraphQL API. Usable for new & existing applications, and auto-scalable.

## Deployment

For the deployment we will follow [these steps](https://www.civo.com/learn/install-argo-cd-in-k3s-civo-cloud-for-deploy-applications#deployment), the only difference is that we will choose in the marketplace the Postgres app

![](https://drive.google.com/uc?id=1_hmQ-PWQ26mF3DbZ6GJsnfPZWloiu9hb)

Once the cluster is running, we will need to download the KUBECONFIG so we can create a user for Hasura and the database. You can do this from the web UI, or using the CLI. The web UI has this option. Save it and set `kubectl` to use it - if this is your only cluster you can save the resulting file as `~/.kube/config`, or if you are running multiple clusters, [follow these instructions](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/).

The first thing will be to connect to the Postgres server within k3s and activate it:

````bash
kubectl run tmp-shell --generator=run-pod/v1 --rm -i --tty --image alpine -- /bin/sh
````

Once inside the pod we will do this:

```bash
apk update
apk add postgresql-client
psql -U $user -h postgresql postgresdb
```

Please note that the `$user` in the above example is given to us us by the Civo UI to connect to the postgres instance so you will need to substitute the generated value for the $user. This will ask us for a password which is given in the UI as well. Once connected to Postgres we run these commands to create a user for hasura and our db also. That user will be given `SUPERUSER` permissions in this way so you can create the schemas that you need within our database:

```sql
CREATE DATABASE todo;
CREATE USER hasura WITH ENCRYPTED PASSWORD 'hasuradb';
ALTER USER hasura WITH SUPERUSER;
```

Once I do this then we will deploy hasura in our cluster of `k3s`:beer: for them we can go to the hasura site in [GitHub](https://github.com/hasura/graphql-engine/tree/master/install-manifests/kubernetes) to download the yaml, in our case I will put them here to make some modifications.

First we will create `deployment.yaml` that will contain this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hasura
    hasuraService: custom
  name: hasura
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hasura
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hasura
    spec:
      containers:
      - image: hasura/graphql-engine:latest
        imagePullPolicy: Always
        name: hasura
        env:
        - name: HASURA_GRAPHQL_DATABASE_URL
          value: postgres://hasura:hasuradb@postgresql:5432/todo
        - name: HASURA_GRAPHQL_ENABLE_CONSOLE
          value: "true"
        - name: HASURA_GRAPHQL_ADMIN_SECRET
          value: supersecretpassword
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
```

then `svc.yml` that will contain this:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hasura-service
spec:
  type: ClusterIP
  ports:
    - name: hasura-port
      port: 8080
      targetPort: 8080
      protocol: TCP
  selector:
    app: hasura
```

and finally our ingress that will be this way:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hasura-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: 64515915-f40a-460f-bea6-bcfa16af1752.k8s.civo.com
    http:
      paths:
      - path: /
        backend:
          serviceName: hasura-service
          servicePort: hasura-port
```

Now only that remains will be to visit our url `64515915-f40a-460f-bea6-bcfa16af1752.k8s.civo.com` which in your case should be modified by those given by the UI of Civo when the cluster was created. Now we will do some tests to see that everything is working fine.

And if all went well then we will see this:

![hasura-2](https://drive.google.com/uc?id=1wSbUZUl1F7Jm5IV6-VdbjRI2bkJxbSAZ)

there we put our password, the one we declared in the `deployment.yaml` and we must load the UI of hasura, which would be this:

![hasura-3](https://drive.google.com/uc?id=1Z0Tztepc8gD1Y3sFiIpCDPMQjjBLK0ip)

Now we go to the `DATA` tab and click on the` SQL` link then in the console that comes out we paste this:

```sql
CREATE TABLE todo_list (id INTEGER PRIMARY KEY, item TEXT, minutes INTEGER);
INSERT INTO todo_list VALUES (1, 'Wash the dishe', 15);
INSERT INTO todo_list VALUES (2, 'vacuuming', 20);
INSERT INTO todo_list VALUES (3, 'Learn some stuff on KA', 30);
```

After this we return to the `GRAPHIQL` tab and we will see that we can already query about our new table, it would look like this:

![hasura-4](https://drive.google.com/uc?id=1HhnI2Pg7yWax-1iokNiCel3b0DLjVO0t)

Well this is a simple example of what you can do with hasura also works for existing databases, I hope it has served them something and it only remains to play with it for a while.