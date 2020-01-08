# Hasura on Civo Cloud

## Introduction

Civo managed Kubernetes is currently in beta, and has a number of applications on the Kubernetes marketplace that can be easily installed on a cluster. At the time of me writing this guide, Hasura is not (yet) available on the marketplace, so I wrote this guide to set up Hasura in the Civo `k3s` service. I hope you find it useful!

If you are not yet a member of the Civo Kubernetes #KUBE100 beta, you can [apply to join here](https://www.civo.com/kube100). All testers get a monthly allowance of $70 for the duration of the beta in exchange for their feedback.

## What is Hasura?

According to the official site:

> Hasura is an [open source](https://github.com/hasura/graphql-engine) engine that connects to your databases & microservices and instantly gives you a production-ready GraphQL API. Usable for new & existing applications, and auto-scalable.

So essentially, Hasura allows us to use GraphQL to query data in databases. That's handy!

## Deployment

As mentioned above, you will need a [Civo](https://www.civo.com) account with Kubernetes access. Once you are logged in, navigate to the Kubernetes menu to create a new cluster. In my case, I am creating a 3-node cluster of Medium-sized nodes (2 CPU, 4GB, 50GB storage each):

![Hasura cluster creation](https://civo-com-assets.ams3.digitaloceanspaces.com/content_images/523.blog.png?1578496393)

When you scroll down to the Marketplace section, select PostgreSQL with a 5GB storage option. Then you are ready to start your cluster!

![Postgresql](https://drive.google.com/uc?id=1_hmQ-PWQ26mF3DbZ6GJsnfPZWloiu9hb)

Once the cluster is running, we will need to download the KUBECONFIG so we can create a user for Hasura and the database. You can do this from the web UI, or using the CLI. The web UI has this option. Save it and set `kubectl` to use it - if this is your only cluster you can save the resulting file as `~/.kube/config`, or if you are running multiple clusters, [follow these instructions](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/).

![Kubeconfig download](https://civo-com-assets.ams3.digitaloceanspaces.com/content_images/526.blog.png?1578496421)

The first thing will be to make sure we set up external access to our database. Create a file called `postgresql-service.yaml` that contains the following:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgresql-service
spec:
  type: LoadBalancer
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
  selector:
    app: postgresql
```

Then, apply it to your cluster with `kubectl apply -f postgresql-service.yaml`.

Next, connect to the Postgres server within k3s and activate it:

````bash
kubectl run tmp-shell --generator=run-pod/v1 --rm -i --tty --image alpine -- /bin/sh
````

Once inside the pod we will do this:

```bash
apk update
apk add postgresql-client
psql -U $user -h postgresql postgresdb
```

Please note that the `$user` in the above example is given to us by the Civo UI. You will need to substitute the generated value for the $user. This will ask us for a password which is given in the UI as well. Once connected to Postgres we run these commands to create a user for hasura in our database as well. That user will be given `SUPERUSER` permissions in this way so you can create the schemas that you need within our database:

```sql
CREATE DATABASE todo;
CREATE USER hasura WITH ENCRYPTED PASSWORD 'hasuradb';
ALTER USER hasura WITH SUPERUSER;
```

You can then exit from the Postgresql administrator by typing `\q` and Enter.

Once this is done, we will deploy Hasura in our `k3s` cluster. The full instructions are on the Hasura repository on [GitHub](https://github.com/hasura/graphql-engine/tree/master/install-manifests/kubernetes). For default values, you can download the yaml file, but in our case I will put them below as I have made some modifications.

First, create a `deployment.yaml` that contains this:

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
          value: supersecretpassword # Change this to your password!
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
```

then `svc.yaml` that will contain this:

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

and finally our `ingress.yaml` that will be along these lines. Please note that you will need to change the `host` line to reflect the DNS entry provided by the Civo Kubernetes UI for your cluster:

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
  - host: 64515915-f40a-460f-bea6-bcfa16af1752.k8s.civo.com # change this!
    http:
      paths:
      - path: /
        backend:
          serviceName: hasura-service
          servicePort: hasura-port
```

## Hasura Usage

Now only that remains will be to visit our cluster URL (in the format `64515915-f40a-460f-bea6-bcfa16af1752.k8s.civo.com` and displayed on the cluster administration page). In this section, we will do some tests to see that everything is working as expected.

And if all went well then we will see this:

![hasura-2](https://drive.google.com/uc?id=1wSbUZUl1F7Jm5IV6-VdbjRI2bkJxbSAZ)

This page will accept our password, the one we declared in the `deployment.yaml` file. Enter it and load the Hasura UI, which should be this:

![hasura-3](https://drive.google.com/uc?id=1Z0Tztepc8gD1Y3sFiIpCDPMQjjBLK0ip)

Now we go to the `DATA` tab and click on the `SQL` link. Then, in the raw SQL console that comes out we paste this and press `Run!`:

```sql
CREATE TABLE todo_list (id INTEGER PRIMARY KEY, item TEXT, minutes INTEGER);
INSERT INTO todo_list VALUES (1, 'Wash the dishes', 15);
INSERT INTO todo_list VALUES (2, 'vacuuming', 20);
INSERT INTO todo_list VALUES (3, 'Learn about k3s on Civo', 30);
```

After this we return to the `GRAPHIQL` tab and we will see that we can already create queries about our new table. Select all relevant items (`id`, `item`, and `minutes`) and hit `Execute Query`. The result should look like this:

![hasura-4](https://drive.google.com/uc?id=1HhnI2Pg7yWax-1iokNiCel3b0DLjVO0t)

## Conclusion

While this is a simple example of what you can do with Hasura once deployed, it also works for existing databases. That means you can use the GraphQL API provided by Hasura to query any SQL database you may have. By adapting the above instructions, you could easily connect an existing database, or play with the one we have created in this exercise to get to grips with Hasura syntax. Let us know what you get up to!

Have you deployed Hasura to query your databases? Let [me](https://www.twitter.com/alejandrojnm) and [@civocloud](https://www.twitter.com/civocloud) on Twitter know!