---
layout: post
title:  "Comparing Yadage and Parsl Workflow Languages"
date:   2019-11-25 11:01:43 +0600
categories: parsl reana HEP SCAILFIN
author: "BenGalewsky"
---
Yadage and Parsl are two popular workflow languages. As part of project [SCAILFIN](https://scailfin.github.io),
we are looking into scripting the REANA reproducible scientific
workflow framework from CERN. It currently supports Yadage and Common
Worklow Language to specify the workflow. We are keenly interested in a possible
role for Parsl in this framework.

## Basics
Yadage and Parsl are both workflow languages and both generate _directed
acyclic graphs_ (DAGs). Yadage represents workflows using YAML and dependencies
are declared in terms of references to other tasks. Parsl uses python to
describe the workflow. It takes more of dataflow view on dependencies. Tasks
dependencies are represented as data result futures. The next task is started
when the dependant data output futures complete.

## Tasks
### Yadage
The atomic unit of the workflow is a packtivity – a packaged activity. It
represents a single parametrized processing step. The parameters a passed as
YAML documents and the processing step is executed using one of multiple
backends. After processing the packtivity publishes JSON data that includes
relevant data for further processing (e.g. referencing files that were created
during the processing). <sup>[1](https://yadage.readthedocs.io/en/latest/definingworkflows.html)</sup>

### Parsl
In Parsl an “app” is a piece of code that can be asynchronously executed on an
execution resource. An execution resource in this context is any target system
such as a laptop, cluster, cloud, or even supercomputer. Execution on these
resources can be performed by a pool of threads, processes, or remote workers.

Parsl apps are defined by annotating Python functions with an app decorator.
Currently two types of apps can be defined: Python, with the corresponding
<code class="highlighter-rouge">@python_app</code> decorator, and Bash, with the corresponding <code class="highlighter-rouge">@bash_app</code> decorator.
Python apps encapsulate pure Python code, while Bash apps wrap calls to external
applications and scripts.<sup>[2](https://parsl.readthedocs.io/en/stable/)</sup>

<div class="html-table">

<table>
<tr>
<th>Feature</th>
<th>Yadage</th>
<th>Parsl</th>
</tr>

<tr>
<td>Task description</td>
<td>Can be a string interpolated command line which are executed as a single bash command. It allows for string interpolation from a parameter set. Can also be a multi-line script which can be executed by bash, python, or Root C++ interpreter.
</td>
<td>Can be a bash script or python function. Python functions are easy; just create a function and decorate @python_app. Bash scripts are also written as decorated python functions that return a string containing the steps to be executed.
</td>
</tr>
<tr>
<td>Parameters</td> <td>A dictionary provided in the <code class="highlighter-rouge"> parameters</code> property of the task. Each of the keys are available for string interpolation in the task's script or command.</td><td> Arguments to the python function can be used directly in python tasks. Can be use in string interpolation for the returned string for the bash steps </td>
</tr>
<tr>
<td>Environment </td> <td>Encoded as part of the task definition. Mostly used to identify the docker image to run the step in as well as some CERN-specific extras to mount a filesystem and provide authentication. Yadage also supports a local process environment which runs on the host server.</td> <td> Not part of the task description and must be associated with the task in the main driver script when setting up the workflow. </td>
</tr>
<tr>
<td> Data Publication </td> <td> Yadage requires a shared filesystem. Data is published by a step using the <code class="highlighter-rouge">publish</code>. There is a <code class="highlighter-rouge">publisher-type</code> property under this which is not well documented. Valid values seen are <code class="highlighter-rouge">frompar-pub</code> which seems to allow for a value to be set as a parameter. <code class="highlighter-rouge">fromglob-pub</code> which expands a directory list from wildcard specified in <code class="highlighter-rouge">globexpression</code>. <code class="highlighter-rouge">constant-pub</code> is another. </td> <td> Parsl functions can have <code class="highlighter-rouge">inputs</code> and <code class="highlighter-rouge">outputs</code> parameters. Each of these are lists of futures. It will wait for the input futures to resolve before starting the task. Likewise, data can be published to the output futures which will not resolve til the task is complete. Parsl has a data management layer that allows for data to be consumed and written to various local and remote filesystems.</td>
</tr>
</table>

</div>





## Workflows
### Yadage
Instead of describing a specific graph of tasks, a yadage workflow definition consists of a collection of stages that describe how an existing graph should be extended with additional nodes and edges. Starting from an empty graph (0 nodes, 0 edges), it is built up sequentially through application of these stages. This allows yadage to process workflows, whose graph structure is not known at definition time (such as workflow producing a variable number of data fragments).

A stage consists of two pieces

A stage body (i.e. its scheduler):
This section describes the logic how to define new nodes (i.e. packtivities with a specific parameter input) and new edges to attach them to the existing graph. Currently yadage supports two stages, one defining a single node and defining multiple nodes, both of which add edges according to the the data accessed from upstream nodes.
A predicate (i.e. its dependencies):
The predicate (also referred to as the stage’s dependencies) is a description of when the stage body is ready to be applied. Currently yadage supports a single predicate that takes a number of JSON Path expressions. Each expression selects a number of stages. The dependency is considered satisfied when all packtivities associated to that stage (i.e. nodes) have a published result<sup>[1](https://yadage.readthedocs.io/en/latest/definingworkflows.html)</sup>

### Parsl
Workflows in Parsl are created implicitly based on the passing of control or data between apps. The flexibility of this model allows for the implementation of a wide range of workflow patterns from sequential through to complex nested, parallel workflows.

Parsl is also designed to address broad execution requirements from workflows that run a large number of very small tasks to those that run few long running tasks. In each case, Parsl can be configured to optimize deployment towards performance or fault tolerance.<sup>[2](https://parsl.readthedocs.io/en/stable/)</sup>

### Workflow Patterns
<div class="html-table">

<table>
<tr>
<th>Pattern</th><th>Description</th><th>Yadage</th><th>Parsl</th>
</tr>
<tr>
<td>Procedural workflows </td><td> Simple sequential or procedural workflows </td><td> Yadage manages the transitions based on outputs from a task completing </td><td> Parsl can track app futures which link end of one task to start of another </td>
</tr>
<tr>
<td>Parallel workflows </td><td> Parallel execution, respecting dependencies among app executions. </td><td> Automatically builds parallel DAGs from spec </td><td> Automatically generated from App dependencies </td>
</tr>
<tr>
<td> Parallel workflows with loops </td><td></td><td> Can be implemented with tasks that produce a variable number of data fragments </td><td> Just a simple python loop that appends calls to the parsl task </td>
</tr>
<tr>
<td> Parallel dataflows </td><td> Parallel workflows driven by data results, and not task completion </td><td> Yadage won't start a new task until all of the outputs from a previous step are complete | Parsl tracks dependencies either by task, or by data future. </td><td>Each Parsl data output is represented as a Future. Tasks can be triggered by one or more of these futures completing</td>
</tr>
</table>
</div>


## Execution Backends
A workflow description language is only as useful as the execution backends.
Both packages offer extensive options for this.

### Yadage
Yadage comes with a command line tool for running workflows. It accepts a
workflow definition and optional parameters. By default it will just run the
workflow as a multiprocessing pool on the local machine.

Yadage can run on various backends such as multiprocessing pools, ipython
clusters, or celery clusters. If human intervention is needed for certain steps,
it can also be run interactively. Significantly, Yadage is supported by CERN's
reproducible science Framework,  Reana and it is used to execute yadage
workflows inside Kubernetes in a  standardized environment.

### Parsl
Parsl scripts can be executed on different execution providers (e.g., PCs,
clusters, supercomputers) and using different execution models (e.g., threads,
pilot jobs, etc.). Parsl separates the code from the configuration that
specifies which execution provider(s) and executor(s) to use. Parsl provides a
high level abstraction, called a block, for providing a uniform description of a
resource configuration irrespective of the specific execution provider.

The parsl ecosystem includes specific interfaces for running on many of the
popular HTC environments such as Jetstream, Condor, and Slurm. It also has
interfaces for commercial cloud providers.

The `HighThroughputExecutor` implements hierarchical scheduling and batching and
consistently delivers high throughput task execution on the order of 1000 Nodes

Key to this performance is the coupling between the parsl driver program and the
executors. It depends on the IPython library in the executor image. The
workflow steps are pickled using `CloudPickle` and transmitted.

## Conclusions
Both frameworks offer flexible and easy to read workflow definitions which are
automatically translated into parallel processing graphs. Both support a number
of execution backends to enable execution of the workflows on a variety of
hosts and clusters available to our community.

The obvious difference between the two is the choice of language used to
represent the workflow. Yadage starts with the popular YAML file format. This
follows a traditional approach that represents workflow using a configuration
file. Parsl's unique approach is to use python as the specification language.

Parsl's unfamiliar "workflow as code" model takes a little getting used to for
new developers, however once the basic concepts are grasped it's very easy
to work with and can express very complicated parallel workflows with ease.
Yadage's YAML files are easy to get started with, but it seems that it would
be difficult to specify anything more complicated than a few parallel threads.

The Parsl HighThroughputExecutor has received notoriety as a high performance
workflow execution back-end and can scale to thousands of nodes. Delivering
this performance requires that Parsl takes control over the execution of the
code and depends on the presence of the parsl library, iPython and Python3
inside any execution environments where it will be run. The versions of these
dependencies must match the version of the driver program.

Yadage execution back-ends take a less invasive approach. They will launch
docker containers and set the command to run as well as any environment
variables. This makes it ideal of reproducible workflows in Reana since there
are very few constraints on what can go into the docker image.

Given the expressiveness of Parsl for complicated parallel workflows and the
highly performant set of executors, Parsl should be considered for demanding
production workflows. However the need to introduce dependencies into the built
images would seem to make it unsuitable for preservation. Yadage is a better
fit for those applications.
