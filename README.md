# Dreams k8s Example

This repository contains example infrastructure as code (IAC) for running the
Dreams Enterprise Stack (DES) with kubernetes (k8s).

## Preparations

To run these examples, you need a k8s cluster.
One way to get one locally quickly is to use [minikube](https://minikube.sigs.k8s.io/docs/).

Follow the [installation steps for your platform](https://minikube.sigs.k8s.io/docs/start/), and start your cluster:

```sh
minikube start
```

To check the progress and status of created resources in this guide, it's
helpful to spin up the minikube dashboard:

```sh
minikube dashboard
```

You also need the DES docker image available locally, tagged as `des:latest`.
Ask Customer Success for access to our Container Registry:

```sh
docker login -u yourusername -p yourpassword quay.io
docker pull quay.io/dreamstech/des:latest
docker tag quay.io/dreamstech/des:latest des:latest
```

Load the image into the minikube cluster:

```sh
minikube image load des:latest
```

## Namespace

Create the `des` namespace

```sh
kubectl apply -f des-namespace.yml
```

## DB

Just for the sake of this example, we spin up a Postgres DB in the k8s cluster,
using a persistent volume to retain data between restarts.

Create a persistent volume and a claim:

```sh
kubectl apply -f des/db/persistent-volume.yml
kubectl apply -f des/db/persistent-volume-claim.yml
```

Deploy Postgres and expose it through a service

```sh
kubectl apply -f des/db/deployment.yml
kubectl apply -f des/db/service.yml
```

Check that the DB is provisioned and reachable.
Create a service tunnel to the minikube cluster, in a separate terminal:

```sh
minikube service -n des db-service --url
```

Assuming that you have `psql` installed, connect to the DB
(replace the port with the above service tunnel output):

```sh
psql -h localhost -U postgres -p <port>
```

The password in this example can be found in [des/db/deployment.yml](des/db/deployment.yml)

You can now verify that you have a working connection, for example by running the `\l` command to list available databases.

## Common Config

Create a common ConfigMap and Secret, which will be used by both the Web Server,
Background Worker containers, as well as one-of Jobs:

```sh
kubectl apply -f des/configmap.yml
kubectl apply -f des/secret.yml
```

## Storage

Instead of using an S3-compatible object storage for DES content, we are using
a file system-based approach by mounting a persistent volume.

All files uploaded to DEPo will be put under `/app/storage` in the DES
containers, and available on the Minikube VM under
`/data/des-storage-persistent-volume`.

```sh
kubectl apply -f des/storage/persistent-volume.yml
kubectl apply -f des/storage/persistent-volume-claim.yml
```

## Prepare the DB

Run a one-of job to create the DB and all its tables:

```sh
kubectl apply -f des/setup-job.yml
```

You can verify that it worked by again connecting to the DB and listing the
tables in the `des` database with the `\dt` command:

```
psql -h localhost -U postgres -p <port> des
Password for user postgres:
des=# \dt
                     List of relations
 Schema |              Name              | Type  |  Owner
--------+--------------------------------+-------+----------
 ...
```

## Web Server

Deploy the web-server and create a service so that it is reachable from outside
the cluster

```sh
kubectl apply -f des/web-server/deployment.yml
kubectl apply -f des/web-server/service.yml
```

Check that the web server is provisioned and reachable.
Create a service tunnel to the minikube cluster, in a separate terminal:

```sh
minikube service -n des web-server-service --url
```

And use a browser to surf to the above command's output. It should give you a
404 page saying "The page you were looking for doesn't exist."

The reason for this error message is that there is no *Partner* defined yet.

## Create a first Partner

Connect to a running web-server pod and execute the `dreams:partner:create` task:

```sh
kubectl -n des exec <pod-name> -- \
  bundle exec rails dreams:partner:create -- \
  --name=acme \
  --organization='Acme Inc.' \
  --subdomain=acme \
  --locales=en \
  --currency=EUR \
  --admin_email=admin@example.com \
  --admin_password=5up3rs3cr3t!
```

Given that `ROOT_DOMAIN` is configured to `dreams.test` in
[des/configmap.yml](des/configmap.yml), and the partner was created with `subdomain=acme`, the
web-server will now serve requests for this partner at the following paths:

- The Dreams Web App: `acme.dreams.test:<port>`
- The DEPo admin panel: `acme.dreams.test:<port>/depo`
- The Dreams API: `acme-api.dreams.test:<port>/v1`

To test this, make sure that you have a service tunnel to the minikube cluster,
in a separate terminal:

```sh
minikube service -n des web-server-service --url
```

and take note on the port number.

Make sure that `acme.dreams.test` and `acme-api.dreams.test` routes to
localhost. Either by installing a DNS resolver like e.g. [dnsmasq](https://formulae.brew.sh/formula/dnsmasq)
or by updating your `/etc/hosts` file with:

```
127.0.0.1   acme.dreams.test
127.0.0.1   acme-api.dreams.test
```

In a real-world setup, the above step corresponds to configuring actual DNS
records.

Now you can
- browse to the Dreams Web App at `http://acme.dreams.test:<port>`; and
- browse to the DEPo admin panel at `http://acme.dreams.test:<port>/depo` and
  logging in with the bootstrap admin credentials given when creating the
  partner; and
- ping the API at `http://acme-api.dreams.test:<port>/v1/ping`.

## Background Worker

To function properly DES also needs a separate container running background
jobs. Create a deployment for it:

```sh
kubectl apply -f des/background-worker/deployment.yml
```

## Next steps

At this point, you should have a fully functional example deployment of DES.
For more information on how to integrate DES with your other systems, harden
the infrastructure, recommended firewall settings etc., see: https://docs.dreams.enterprises/docs/self-hosted/architecture-overview.
