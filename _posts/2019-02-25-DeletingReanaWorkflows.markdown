---
layout: post
title:  "Deleting Reana Workflows"
date:   2019-02-25 11:01:43 +0600
categories: parsl reana HEP SCAILFIN
author: "BenGalewsky"
---
Until Reana 0.5.0 comes out, there is no way to clear out workflows. This
becomes a bit of a mess during testing. Here is how to clear them out
manually in the mean time:

From a node that has `kubectl` access to the cluster, find the pod running
postgres and open up a shell:
```bash
kubectl get pod | grep ^db
kubectl exec -it <<<pod name>> bash
```

Inside the pod start up the postgres SQL command line:
```bash
psql --username $POSTGRES_USER --dbname $POSTGRES_DB
```

Finally truncate the workflow table:
```sql
truncate table workflow;
```

Now we need to delete the files associated with these workflows. Exit the
database pod and now find the reana server pod and start a shell there:
```bash
kubectl get pod | grep ^serv
kubectl exec -it <server pod> bash
```
Go to the `/reana/users` directory, and under each user, clearn out the contents
of the `workflows` directory

Exit the shell and verify that the workflows are gone with your reana-client:
```bash
reana-client workflows
NAME   RUN_NUMBER   CREATED   STATUS
```

Go forth and create new workflows!

This work has been supported by the NSF and Project [SCAILFIN](https://scailfin.github.io)
