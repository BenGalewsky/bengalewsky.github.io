---
layout: post
title:  "Building Tensorflow Docker Image for Openstack"
date:   2018-12-10 22:01:43 +0530
categories: openstack docker
author: "BenGalewsky"
---
With the many layers of virtualization, it's easy to forget that docker images
ultimately run on a real CPU with its own architecture and characteristics. This
is usually not an issue, except when the code in the image makes assumptions about
that CPU. The Tensorflow library makes use of whatever hardware acceleration
is available. It is quite sensitive to the hardware the image is built on.

I discovered this when trying to deploy a colleague's Tensorflow application on
our OpenStack cluster. I was using the `tensorflow/tensorflow` Docker image.
When I attempted to run it, it simply printed out the error:

```
Illegal instruction     (core dumped) python3 ./${MAIN_SCRIPT}
```

This is caused by the standard images being built to take advantage of the AVX2 instruction
set. Since Tensorflow 1.6 the pre-built binaries assume the CPU supports those instructions.

This post describes how I compiled Tensorflow for the specific hardware backing
our OpenStack and deployed my application via docker.

# Build Tensorflow Locally
Tensorflow is a big and complicated system. Compiling it depends on a large number
of dependencies and specialized build tools. Fortunately, the community has created a handy 
dockerized process to help with this build.

- Create a VM on the OpenStack cluster. I used an Ubuntu 16.0.2 image.
- Install:
  + git
  + docker
  + docker-compose
- Make sure the ubuntu user has
[permission to run docker commands](https://docs.docker.com/install/linux/linux-postinstall/#manage-docker-as-a-non-root-user).
- Log into the VM and checkout a copy of my fork of Hadrian Mary's [tensorflow-builder](https://github.com/BenGalewsky/docker-tensorflow-builder)
- For now use my `ncsa_openstack` branch. [Pull Request](https://github.com/hadim/docker-tensorflow-builder/pull/13) is in progress
- As per the [README](https://github.com/hadim/docker-tensorflow-builder/blob/master/README.md)...

```bash
cd tensorflow/ubuntu-16.04/

# Build the Docker image
docker-compose build

# Set env variables
export PYTHON_VERSION=3.5
export TF_VERSION_GIT_TAG=v1.9.0
export USE_GPU=0

# Start the compilation
docker-compose run tf

# You can also do:
# docker-compose run tf bash
# bash build.sh
```

- Wait, wait, and wait some more... It's a very long build time!
- The wheel file for your architecture is sitting in the repositories wheels/ folder

# Create a Docker Image
As a final step, I created a Dockerfile based on a Python image which installs the Tensorflow wheel:

```dockerfile
FROM python:3.5-stretch
COPY tensorflow-1.9.0-cp35-cp35m-linux_x86_64.whl /
RUN  pip install /tensorflow-1.9.0-cp35-cp35m-linux_x86_64.whl
CMD ["python3"]
```


