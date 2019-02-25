---
layout: post
title:  "Porting Reana Atlas Recast Demo to Parsl"
date:   2019-02-25 15:01:43 +0600
categories: parsl reana HEP SCAILFIN
author: "BenGalewsky"
---

As part of my work on project [SCAILFIN](https://scailfin.github.io) I am
exploring how we could express [Parsl](https://parsl.readthedocs.io/en/latest/)
workflows as [Reana](http://www.reana.io) reproducible research objects. Parsl
is a parallel workflow system that is expressed as code. Dan Katz published
an [informative article](https://danielskatzblog.wordpress.com/2019/02/05/using-workflows-expressed-as-code-and-workflows-expressed-as-data-together/)
describing the motivation behind this approach.

To learn more about parsl and to see how it would fit into the reana world, we
decided that I should port one of the reana demos from yadage to parsl. I did
this with the
[Atlas Recast Demo](https://github.com/BenGalewsky/reana-demo-atlas-recast).

You can see the full example in the `parslization` branch of
[my fork of the demo](https://github.com/BenGalewsky/reana-demo-atlas-recast/tree/parslization)

# Basic Approach
The demo is implemented as two steps, each relying on a docker image based on
the [Atlas Standalone Analysis Image](https://hub.docker.com/r/atlas/analysisbase).
This image provides many of the basic tools needed for working with events in
ATLAS Root files.

The first step downloads a root file and extracts specific events into an output
file. The second step uses the extracted events and produces some plots. These
two steps are implemented in parsl as parameterized bash scripts running in the
two docker containers.

For our purposes we will be running these two containers on a kubernetes cluster
with parsl controlling them.  The two steps interact through a shared persistent
volume.

![Parsl Workflow](https://github.com/BenGalewsky/bengalewsky.github.io/raw/master/images/atlas-recast-in-parsl.png)

We tell parsl to orchestrate the two containers through the generated file. The
first container starts, and the second container waits till the file is
generated.

# Parsl Control of Containers
Parsl interacts with Docker containers through an `ipython` engine. For this to  
work, the container must have python3 with the parsl package installed. This
proved to be the most vexing part of the whole project. The Atlas `analysisbase`
image is based on a Centos6 image which only has Python2.7 available.

The only way I could figure out how to get this environment installed in the
image without breaking dependencies that some of the earlier steps in the
Docker build rely on was to add a final step to Docker file that actually
updates the development tools and builds python3 from source.

``` Dockerfile
RUN yum update -y && \
    yum groupinstall "Development Tools" -y && \
    wget https://www.python.org/ftp/python/3.6.8/Python-3.6.8.tgz && \
    tar -xvf Python-3.6.8.tgz && \
    cd Python-3.6.8 && \
    ./configure && \
    make && \
    make altinstall && \
    /usr/local/bin/pip3.6 install cython && \
    /usr/local/bin/pip3.6 install parsl
```

In my fork of the Atlas demo I updated the two Dockerfiles. I built them and
deployed them to [my DockerHub](https://cloud.docker.com/repository/list) repo.

# Creating Persistent Volume
The two steps interact with each other and publish their outputs using a
persistent volume that is shared between them. For simplicity's sake, I tested
all of this out using [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
with a host-mounted volume.

To start with, I created a directory inside minikuibe's VM by
``` bash
minikube ssh
```
and then
```bash
sudo mkdir /mnt/data
```

Back on my own workstation I created the persistent volume as `pv.yaml`:
``` yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/data"
```

and a presistent volume claim as `pvc.yaml`:
``` yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
```

Then created them in cluster with:
```
kubectl create -f pv.yaml
kubectl create -f pvc.yaml
```

We will need some files from the repo to be copied to this persistent volume.
One of the features of reana is that it handles the copying of data into the
cluster. Since I'm not using reana for this example, I launched a busybox pod
in the cluster with the volume mounted with
`debug-pod.yaml` as:
```
kind: Pod
apiVersion: v1
metadata:
  name: volume-debugger
spec:
  volumes:
    - name: volume-to-debug
      persistentVolumeClaim:
       claimName: task-pv-claim
  containers:
    - name: debugger
      image: busybox
      command: ['sleep', '3600']
      volumeMounts:
        - mountPath: "/data"
          name: volume-to-debug
```

I then created the pod and opened a shell:
``` bash
kubectl create -f debug-pod.yaml
kubectl exec -it volume-debugger sh
```

Inside that volume-debugger I created a `/data/code` directory and then from
my workstation I copied the python scripts from the atlas-recast repo into that
directory:
``` bash
cd statanalysis
kubectl cp make_ws.py volume-debugger:/data/code/make_ws.py
kubectl cp plot.py volume-debugger:/data/code/plot.py
kubectl cp set_limit.py volume-debugger:/data/code/set_limit.py
kubectl cp data/background.root volume-debugger:/data/background.root
kubectl cp data/data.root volume-debugger:/data/data.root
```

I confirmed that everything was where I expected with the volume-debugger shell.

# Configuring Parsl Executors for Kubernetes
Parsl runs the workflow steps using executors which are linked to specific
providers which run the code. We will be using the `IPyParallelExecutor` and
the `KubernetesProvider`. The provider's configuration is where we specify the
docker image to use. It also allows for persistent volumes to be mounted into
the containers.

Here is the config object that I use:
``` python
config = Config(
    executors=[
        IPyParallelExecutor(
            label='event_selection',
            provider=KubernetesProvider(
                image="bengal1/reana-demo-atlas-recast-eventselection:latest",
                nodes_per_block=1,
                init_blocks=1,
                max_blocks=1,
                parallelism=1,
                persistent_volumes=[("task-pv-claim","/data")]
            )
        ),
        IPyParallelExecutor(
            label='stat_analysis',
            provider=KubernetesProvider(
                image="bengal1/reana-demo-atlas-recast-statanalysis",
                nodes_per_block=1,
                init_blocks=1,
                max_blocks=1,
                parallelism=1,
                persistent_volumes=[("task-pv-claim", "/data")]
            )
        )
    ],
    lazy_errors=False
)
```
This defines an Executor for each step along with the Docker image which
implements the behavior. Each executor is given a name which is referenced
in the parsl steps.

# The Workflow
Now we get into the parsl workflow as code! There are two steps in our workflow
each are bash scripts that run on their Docker container. The key is the
`@App` annotation:
```
@App('bash', executors=['stat_analysis'], cache=True)
```

Since the second step depends on a file produced by the first step, we set up
a data dependency. This is done by creating a `parsl.File` handle to the
generated hist-sample.root file. When the first step is invoked, a future of
this file is made available. I made it an input data dependency to the second
step.

This step in turn produces a data future for a .png plot that it produces. I run
the workflow, by invoking both steps and then waiting for the final result data
future to to resolve.

``` python
event_selection_future = event_selection("404958", "recast_sample", 0.00122,
                                         dxaod_file="https://recastwww.web.cern.ch/recastwww/data/reana-recast-demo/mc15_13TeV.123456.cap_recast_demo_signal_one.root",
                                         submitDir="/data/submitDir",
                                         lumi_in_ifb=30.0,
                                         outputs=[sample_file])

stat_analysis_future = stat_analyis("/data/data.root", "/data/submitDir/hist-sample.root",
                           "/data/background.root", "/data/results",
                                    inputs=event_selection_future.outputs,
                                    outputs=[post_file])


# Check if the result file is ready
print ('Done: %s' % stat_analysis_future.outputs[0].done())
print('result is '+str(stat_analysis_future.outputs[0].result()))
```
# Conclusions
This exercise was a great opportunity to learn more about parsl as well yadage
and a chance to think a bit more about the features of reana. My first reaction
is that I enjoyed parsl's _workflow as code_ model. I found it much easier to
learn and read than yadage's yaml model.

One disadvantage I found was that due to parsl's reliance on iPython for
execution it winds up being more invasive to the deployed Docker containers.
The requirement for the parsl package and Python3 in the image came close to
being a show stopper for the legacy code I was orchestrating.

The other thing I noted after running this demo under reana is just how
convenient reana's handling of file deployments is. They offer a command line
tool where you can upload files into a volume the cluster which will be mounted
into running containers. It completely handles the persistent volumes itself.
Parsl has some great tools for copying files into some other environments. I
smell a nice parsl pull request coming on to support transferring files in and
out of a persistent volume in the cluster.


This work has been supported by the NSF and Project [SCAILFIN](https://scailfin.github.io)
