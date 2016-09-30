# Elastic Workers for Spark in Oshinko

## Introduction

Describes modifications and enhancements to add elastic worker logic to Oshinko
Spark clusters.  This solution is driven primarily by the information
available from Spark's published metrics, and Spark cluster information
stored by Oshinko itself.

## Problem statement

The purpose of this enhancement is to allow Spark clusters created in Oshinko
to manage their own compute scale by either increasing or decreasing their
number of Spark worker pods, based on the number of Executors being requested
by Spark's dynamic Executor feature.

## Proposed solution

The proposed solution involves the following modifications:

* Add containers and configuration to the Spark Master pod for Spark metric
sinks.  In particular, carbon and graphite containers.   This modification
is proceeding as a separate Spec and PR, and so the carbon/graphite work
can be considered as a "blocker" for this elastic worker enhancement.
* Add configuration settings to Spark to enable dynamic Executors.  This can
be accomplished either by modifying the Spark Dockerfiles to add this
configuration to `spark_defaults.conf`, or it can be done by adding the
configurations to the `SPARK_MASTER_OPTS` and `SPARK_WORKER_OPTS` environment
variables, which can be done via the template.
* Add another new container to the Spark master pod that runs a Python script
which can poll the graphite REST endpoint and the Oshinko REST endpoint
and use this information to instruct Oshinko to increase or decrease the
number of worker pods.  This may warrant its own repo, with Dockerfile,
Makefile, etc.
* Addition of SPARK_WORKER_CORES and SPARK_WORKER_MEMORY environment variables
to allow control of the resource size of each worker.  This may need to be
synced with corresponding limits configured for the pods themselves, so that
pods are not over-subscribed on physical machines by Openshift.

### Alternatives

We are examining a possible "kube-native" solution being developed at
Google, which could be considered a competitor solution.  This solution
has not been rejected yet for any reason, but is also in prototype stage.
At the time of this submission, the prototype code from Google resides
here:
https://github.com/foxish/spark/tree/k8s-support

We also prototyped a solution that modified Spark core with ability to detect
scaling plug-ins.  This solution requires modifications to the spark upstream,
but does not provide any value over and above the proposed non-invasive
solution, and so it is no longer being considered.

## Affected Components

The primary affected component is the Spark cluster template file.

## Testing

Testing probably is going to be system/integration level.   It is relatively
easy to set up a Spark job that runs long enough to trigger increasing
Executor requests and then manually observe the number of Spark worker pods
as they increase and decrease.

At the present time I am not sure how to best automate that kind of test.

## Documentation

Any additional parameters to the template will need to be documented.
In my prototype I have parameters for the docker image for the elastic
python container.  Additional parameters could be settings for
SPARK_WORKER_CORES and SPARK_WORKER_MEMORY.  Whether or not to enable
elastic workers at all could also be made a parameter.   Other possible
parameters could be minimum workers and maximum workers.
