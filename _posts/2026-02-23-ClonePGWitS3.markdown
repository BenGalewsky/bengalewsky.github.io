---
layout: post
title:  "Migrating a Postgres Database Across Clusters"
date:   2026-02-23 15:55:00 +0600
categories: kubernetes
author: "BenGalewsky"
---
If you need to migrate a postgres database from one Kubernetes cluster to another you need
a strategy for copying the data to a shared storage area. This posting describes how to 
do this using a S3 bucket. In particular it has a few hard-won hints for making this work
with an [Open Storage Network](https://www.openstoragenetwork.org/) bucket.

## Create a BusyBox Pod in the Source Cluster
Here's a manifest for creating a busy box pod, with secrets loaded from the common
Postgres operator:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-busybox
  namespace: airglow
spec:
  containers:
  - command:
    - bash
    env:
    - name: AWS_REQUEST_CHECKSUM_CALCULATION
      value: when_required
    - name: AWS_RESPONSE_CHECKSUM_VALIDATION
      value: when_required
    image: postgres:17
    imagePullPolicy: IfNotPresent
    name: postgres-client
```

Get a bash shell into the busybox
```bash
kubectl exec -it postgres-busybox -- bash
```

It's convinient to use the postgres docker image, but it won't have the tools
to interact with S3, so let's install them now

```bash
apt-get update && apt-get install -y awscli
export AWS_ACCESS_KEY_ID=<MY KEY>
export AWS_SECRET_ACCESS_KEY=<MY SECRET>
export AWS_ENDPOINT_URL=https://rice1.osn.mghpcc.org # URL of your OSN endpoint
# Some options to make AWS tools work with OSN flavor of S3
export AWS_REQUEST_CHECKSUM_CALCULATION=when_required
export AWS_RESPONSE_CHECKSUM_VALIDATION=when_required
aws s3 ls my-bucket
```

# Make the Backup and Save in Bucket
Use pg_dump to make a backup to the temp folder:
```bash
pg_dump -h postgres-cluster-ro -U software_user software -F c -f /tmp/dump.backup
aws s3 cp  /tmp/dump.backup s3://my-bucket/dump.backup
```

# Create a Friendly BusyBox in the New Cluster
I now like to create a comprehensive secret in the namespace of new clusters that make it easy to provide
environment variables for connecting to the db as well as the remote bucket.


```yaml
apiVersion: v1
kind: Secret
metadata:
  name: software-user-credentials
  namespace: dagster
type: kubernetes.io/basic-auth
stringData:
  username: software_user
  password: "XxxXXxxXxX"
  uri: postgresql://software_user:xXXXxxxXX@postgres-cluster-rw.dagster:5432/software

  port: "5432"
  dbname: software
  host: postgres-cluster-rw

  AWS_ENDPOINT_URL: "https://ncsa.osn.xsede.org"
  AWS_ACCESS_KEY_ID: <My key>
  AWS_SECRET_ACCESS_KEY: <My secret>
```

Now create the busy box pod with these secrets available as environment variables:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-busybox
spec:
  containers:
  - name: postgres-client
    image: postgres:17
    command:
    - bash
    env:
    - name: DB_URI
      valueFrom:
        secretKeyRef:
          name: software-user-credentials
          key: uri
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: airglow-user-credentials
          key: host

    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: airglow-user-credentials
          key: username

    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: airglow-user-credentials
          key: password

    - name: DB_DATABASE
      valueFrom:
        secretKeyRef:
          name: airglow-user-credentials
          key: dbname

    - name: AWS_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: software-user-credentials
          key: AWS_ACCESS_KEY_ID
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: software-user-credentials
          key: AWS_SECRET_ACCESS_KEY
    - name: AWS_ENDPOINT_URL
      valueFrom:
        secretKeyRef:
          name: software-user-credentials
          key: AWS_ENDPOINT_URL
    - name: AWS_REQUEST_CHECKSUM_CALCULATION
      value: "when_required"
    - name: AWS_RESPONSE_CHECKSUM_VALIDATION
      value: "when_required"

    stdin: true
    tty: true
  restartPolicy: Never
  ```
  
  Now you can easily get a psql prompt with 
  ```bash
  kubectl exec -it postgres-busybox -- sh -c 'psql $DB_URI'
  ```
  
  # Load the Dump Into the New Database
  We now have everything in place to load the dump into the new database.
  
  Create a shell into the busybox pod and install the AWS CLI
  
  ```bash
  kubectl exec -it postgres-busybox -- bash
  apt-get update && apt-get install -y awscli
  
  aws s3 ls s3://my-bucket
  ```
  
  You should see your backup dump.
  
  Now let's copy it down to /tmpo
  ```bash
  aws s3 cp s3://my-bucket/dump.backup /tmp/dump.backup
  ```

And load the dumped data into our database
```bash
echo $DB_PASSWORD
pg_restore -h $DB_HOST -U $DB_USER -v -d $DB_DATABASE /tmp/dump.backup
```
Take a look around and verify that the data is loaded

```bash
psql $DB_URI
\dt
```

If you are reloading on top of an existing restore, then add `--clean` to the pg_restore command 
to drop things before starting.
