---
layout: post
title:  "Securing Traefik Ingress with BasicAuth"
date:   2026-02-15 09:35:00 +0600
categories: kubernetes
author: "BenGalewsky"
---
Sometimes you just want to deploy a kubernetes app which needs to be secured
with some sort of auth, but you don't need to fuss with an entire OAuth workflow.

This is very easy to do with traefik ingress controller.

This will require a Kubernetes opaque secret that contains an htpasswd style
entry. Then a traefik `Middleware` that points to that secret

## Create Your Auth String
Use the cli tool `htpasswd` to create a password file:

```bash
htpasswd -c <file to generate> <username>
```

It will prompt you for a password and then generate a file the
file which will have a single line containing the encoded username
and password.

## Create the Secret
You will create a specifically formatted secret to be consumed by traefik:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: dagster-basic-auth-secret
  namespace: dagster
type: Opaque
stringData:
  auth: |
    myuser:$apr1$encodedpasswordstuff
```

Add this secret to your cluster. Keep track of the password you used
since you can't get it back from this string.

## Create the Middleware
Now we create the trafefik middleware:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth  # Will be named dagster-basic-auth inside traefik
  namespace: dagster
spec:
  basicAuth:
    secret: dagster-basic-auth-secret
```
Traefik does a weird thing to namespace these global middleware objects.

You will use this in your ingress definition. Since the middlewares are global,
you put the namespace in front of the middleware defination name to
get traefkik to find it.

```yaml
ingress:
  enabled: true
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: dagster-basic-auth@kubernetescrd
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
```

## In Use
Now when you visit the URL backed by the ingress, you will get the usual
BasicAuth prompt. Enter the user name and password and you will be 
granted access.
