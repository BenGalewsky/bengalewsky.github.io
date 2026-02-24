---
layout: post
title:  "CloudNative Postgres Operator Best Practice"
date:   2026-02-24 15:55:00 +0600
categories: kubernetes
author: "BenGalewsky"
---
The [CloudNative postgres Operator](https://cloudnative-pg.io/docs/1.28/) is a terrific tool 
for deploying production quality postgres instances. We've found it to be far superior to 
to the bitnami helm chart (which we have to drop anyway due to loss of support).

The operator adds Custom Resource Definitions to the cluster so you can create a postgres deployment
as well as databases using declarative yaml.

# Create a Postgres Cluster
Get started by create a postgres cluster. This is a custom resource definition that the operator will watch and create a postgres cluster for you. Here's an example manifest:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: dagster
spec:
  enableSuperuserAccess: true  # Do this to enable remote login
  instances: 3

  # Storage configuration using csi-cinder-sc-retain storage class
  storage:
    storageClass: csi-cinder-sc-retain
    size: 50Gi

  # PostgreSQL configuration
  postgresql:
    parameters:
      shared_buffers: "256MB"
      max_connections: "50"
      work_mem: "4MB"

  # Resources allocation
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1"

  managed:
    roles:
    - name: software_user
      ensure: present
      comment: Software Directorate DB User
      login: true
      superuser: false
      inRoles:
        - pg_monitor
        - pg_signal_backend
      passwordSecret:
              name: software-user-credentials

```
This will create a postgres cluster with 3 instances, using the cinder storage class. 

This is a good production quality configuration with full replication. On the Radient cluster 
we need to specify the cinder storage class since the default storage class is based on NFS which is not suitable for postgres.

We need to create the roles (users) for any database we add to this cluster. We typically put the
yaml for the database in a different file, but the role definition needs to be in the same namespace as the cluster. The password for the user is stored in a secret that you need to create before applying this manifest.

The password for the postgres admin user is automatically generated and stored in a secret named `<cluster-name>-superuser` in the same namespace as the cluster. You can get the password with the following command:

```bash
kubectl get secret postgres-cluster-superuser -n dagster -o jsonpath="{.data.password}" | base64 --decode
```

Even better is that the operator adds a property to the secret that is the full URI that can be used to connect to the database. You can get that with the following command:

```bash
kubectl get secret postgres-cluster-superuser -n dagster -o jsonpath="{.data.uri}" | base64 --decode
```

# Adding a Database
As we've seen, we need to create the role for any database we want to add. The role is authenticated
with a password that is stored in a secret. I like to create a comprehensive secret that contains all the environment variables needed to connect to the database as well as the bucket where we will store backups. Here's an example manifest for creating that secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: software-user-credentials
  namespace: dagster
type: kubernetes.io/basic-auth
stringData:
  username: software_user
  password: "%XXXXXXX6"
  uri: postgresql://software_user:%XXXXXXX6@postgres-cluster-rw.dagster:5432/software

  port: "5432"
  dbname: software
  host: postgres-cluster-rw

  AWS_ENDPOINT_URL: "https://ncsa.osn.xsede.org"
  AWS_ACCESS_KEY_ID: <My ACCESS KEY>
  AWS_SECRET_ACCESS_KEY: <My SECRET KEY>
```

Now we can create the database with the following manifest:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: software
  namespace: dagster
spec:
  cluster: postgres-cluster
  owner: software_user
```

# Declarative Updates
The nice thing about using the operator is that you can make updates to the cluster and database by simply updating the yaml and applying it. For example, if you want to add a new database, you just create a new manifest for that database and apply it. If you want to change the configuration of the cluster, you just update the cluster manifest and apply it. The operator will take care of making the necessary changes to the cluster and databases.
