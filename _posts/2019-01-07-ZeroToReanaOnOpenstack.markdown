---
layout: post
title:  "Zero to Reana on Openstack"
date:   2019-01-24 11:01:43 +0530
categories: openstack reana HEP SCAILFIN
author: "BenGalewsky"
---

[Reana](http://www.reana.io) is a reproducible research data platform based on
Kubernetes that is being developed at CERN. The team have developed a useful
Python script for installing and running the platform. It's only tested with
a local minikube installation and a cloud-based installation on CERN's
_lxplus-cloud_ OpenStack.

I was asked to set up an instance on NCSA's OpenStack cluster called Nebula.
This posting describes the step-by-step recipe I followed. It should work for
any generic OpenStack, including [Jetstream](https://portal.xsede.org/jetstream)
at Indiana or TACC.

# Overview of the Process
For this recipe we will:
1. Provision the cluster in a private network using Terraform
2. Install Kubernetes using the same Terraform proces
3. Install a Traefik ingress and secure with LetsEncrypt
4. Create a persistent volume for the reana resources
5. Install reana
6. Run "Hello, World"!

# Provision the Cluster
In our office, we have created a robust process for provisioning a Kubernetes
cluster on OpenStack. It uses Terraform along with the simple and reliable
[KubeADM Bootstrap](https://github.com/data-8/kubeadm-bootstrap) developed
by the Data8 group at Berkeley.

We further automated this process by incorporating it into a complete
[Terraform](https://www.terraform.io) spec for provisioning the hosts, network,
and attached storage.

1. Install Terraform
2. Check out a copy of [https://github.com/nds-org/kubeadm-terraform](https://github.com/nds-org/kubeadm-terraform)
3. Download credentials from your OpenStack dashboard. This comes as shell script
which sets all of the environment variables needed for Terraform to
communicate with your OpenStack api. You need to `source` this script and enter
your OpenStack password when requested.
4. Create a cluster configuration .tfvars file in a a new directory called
`config` in your copy of the repo:
```
env_name = "reana"
pubkey = "~/.ssh/cloud.key.pub"
privkey = "~/.ssh/cloud.key"
master_flavor = "m1.large"
image = "Ubuntu 16.04"
worker_flavor = "m1.large"
public_network = "ext-net"
worker_count = "2"
worker_ips_count = "1"
docker_volume_size = "75"
storage_node_count = "1"
storage_node_volume_size = "50"
dns_nameservers = ["141.142.2.2", "141.142.230.144"]
external_network_id="xxxx-x-x-xxx-x-xxx"
pool_name="ext-net"
```
You will need to customize this to your needs. Take a look at the [README](https://github.com/nds-org/kubeadm-terraform/blob/master/README.md) for
specifics of these variables. Basically, you need one or more worker nodes, an
external IP address, and a storage node to host persistent volumes.

6. Initialize Terraform plugins:
```
terraform init
```

7. Apply the terraform spec
```
terraform apply --var-file=configs/myConfig.tfvars
```

5. You now have an empty kubernetes cluster! For security purposes the api
is not accessible from the outside world. You need to ssh into the master
node. It has `kubectl` installed so you can start interacting with
your cluster from there.
```
ssh -i ~/.ssh/cloud.key ubuntu@xxx.xx.xxx
```


6. The kubeadm process installs a helm chart called `support` which includes an
ingress controller which would conflict with the traefik controller Reana
requires. We'll just delete this chart.
```
helm delete support
```

# Install Traefik Ingress Controller
The ingress controller sits on the node that has an external IP address and
routes inbound http requests to the reana service regardless of the node
where it is running.

With helm, installation is quite simple. However, I wound up forking the helm chart as
I couldn't see how the node selector settings worked as implemented to insure
the controller runs on the node that has an external IP address.

1. We tell helm about my repo where the forked chart is served:
```
sudo helm repo add openstack-traefik https://bengalewsky.github.io/openstack-traefik
```
2. Use the OpenStack dashboard to look up the Private Network IP and the
public IP of the node that will serve as your gateway.
3. Update your DNS record to add a wildcard `A` record that points to the external
IP address of this ingress node.
4. Deploy traefik as:
```
sudo helm install openstack-traefik/traefik --name traefik-ingress-controller \
    --namespace kube-system --set serviceType=NodePort \
     --set externalIP=192.168.0.3 --set rbac.enabled=true \
     --set acme.logging=true \
     --set ssl.enabled=true,ssl.enforced=true,acme.enabled=true
     --set acme.email=me@illinois.edu \
     --set acme.challengeType=http-01 \
     --set acme.staging=false
```
Let's unpack this... Set the `externalIP` to be the _private IP_ of your ingress
node. (It's called `externalIP` to distinguish it from the IP inside the
Kubernetes overlay network). We are turning encryption with
[LetsEncrypt](https://letsencrypt.org). Traefik will coordinate with that service
to generate certificates for each ingress domain. During debugging, you might
set `acme.staging` to true. Traefik will ask for staging certs which look
self-signed, but don't count against your rate-limited requests to LetsEncrypt.

5. I used [this example](https://github.com/nginxinc/kubernetes-ingress/tree/master/examples/complete-example)
to validate the traefik installation with a trivial ingress and service.

# Create a Persistent Volume Claim
Reana makes extensive use of a shared persistent volume to hold the service
together. One of the CERN-isms embedded in the Reana cluster init process is
around storage. You have the choice of deploying as a single node development
cluster, or using a hard-coded persistent volume claim named `manila-cephfs-pvc`.

Create a PVC config file, `pvc.yaml` as:
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: manila-cephfs-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```
and then tell kubernetes to create the pvc:
```
kubectl create -f pvc.yaml
```

# Install Reana Cluster
There is a python script provided by the Reana team that will deploy the system
into the current cluster. Ideally some day this should be migrated to Helm.. might be a nice pull
request to contribute...

```
virtualenv reana
source reana/bin/activate
pip install reana-cluster
```

Get a copy of the reana cluster config from this deployment:
`~/.virtualenvs/reana/local/lib/python2.7/site-packages/reana_cluster/configurations/reana-cluster.yaml`


Two important changes for this file:
1. Add a line at the `cluster:` level of the file:
```
reana_url: "reana.yourdns.name.org"
```
This will be the URL for your reana service.
2. In the `environment` property under `reana-job-controller` set:

```
- REANA_STORAGE_BACKEND: "CEPHFS"
```
This will tell the job controller to create job pods that mount our shared
volume instead of relying on non-shared local storage.

4. Deploy reana as:
```
reana-cluster -f reana-cluster.yaml --prod init
```
The `--prod` option says to provision the system with the shared persistent
volumes instead of just local host mounts.

5. The deployment creates an ingress rule that assumes we are providing the certs
ourselves. Edit the ingress:
```
kubectl edit ingress frontend
```
and remove these lines:
```
tls:
  - secretName: reana-ssl-secrets
```

6. Traefik will contact LetsEncrypt and obtain some nice valid certs for this
domain.
7. Test out connectivity with:
```
curl -k https://reana.yourdns.name.org/api/ping
```

# Install Client and _Hello, Reana_
On your local workstation we are finally going to install the Reana client and
run a simple workflow on the cluster.

You need to set environment variables to tell the client how to connect to
your cluster.

On the kubernetes master node, issue this command
```
reana-cluster env --include-admin-token
```

There is a [bug](https://github.com/reanahub/reana-cluster/issues/73)
in the reana-cluster cli where it won't tell us the correct url for our cluster,
but it can easily figure that out ourselves. Do make a note of the `REANA_ACCESS_TOKEN`

1. On your local workstation clone the [reana-demo-helloworld](https://github.com/reanahub/reana-demo-helloworld)
repository.
2. Follow the instructions to install the reana client
```
virtualenv ~/.virtualenvs/myreana
source ~/.virtualenvs/myreana/bin/activate
pip install reana-client
```
3. Set the evironment variables. The `REANA_ACCESS_TOKEN` can be set with the
output from the `reana-cluster` command run previously. Set the `REANA_SERVER_URL`
as
```
export REANA_SERVER_URL=https://reana.yourdns.name.org
```
4. From the root of the hello world repo create your analysis
```
reana-client create -n my-analysis
```
5. Set the `REANA_WORKON` environment var so you don't have to specify
it on every reana-client invocation.
```
export REANA_WORKON=my-analysis
```
6. Upload the python code and the datafile to the cluster.
```
reana-client upload ./data ./code
reana-client list
```
7. Run the workflow!
```
reana-client start
reana-client status
```
8. Download the generated files to your workstation
```
reana-client list
reana-client download results/greetings.txt
```

# Next Steps
Congratulations! You have running Reana cluster with secure remote access.
Review the documentation at [https://reana.readthedocs.io/en/latest/](https://reana.readthedocs.io/en/latest/)

You can also join the [gitter](ttps://gitter.im/reanahub/reana) to interact with the development commuity.

This work has been supported by the NSF and Project [SCAILFIN](https://scailfin.github.io)
