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
sinks (an example metric system is carbon/graphite, as used in the
initial prototype, however the particular metric system is TBD).
* Add configuration settings to Spark to enable dynamic Executors.  This can
be accomplished either by modifying the Spark Dockerfiles to add this
configuration to `spark_defaults.conf`, or it can be done by adding the
configurations to the `SPARK_MASTER_OPTS` and `SPARK_WORKER_OPTS` environment
variables, which can be done via the template.
* Add another new container to the Spark master pod that runs a process
which can poll the graphite REST endpoint and the Oshinko REST endpoint
and use this information to instruct Oshinko to increase or decrease the
number of worker pods.
* Addition of SPARK_WORKER_CORES and SPARK_WORKER_MEMORY environment variables
to allow control of the resource size of each worker.  This may need to be
synced with corresponding limits configured for the pods themselves, so that
pods are not over-subscribed on physical machines by Openshift.

### Alternatives

There is an upstream project to add native support for kubernetes to Apache
Spark.  At the time of this writing, the prototype code lives here:
https://github.com/foxish/spark/tree/k8s-support
The kubernetes issue:
https://github.com/kubernetes/kubernetes/issues/34377
The Apache Spark JIRA:
https://issues.apache.org/jira/browse/SPARK-18278

The upstream kube-native project will not be available in the near term, but is
likely to be a solution of choice when it becomes available upstream.

We also prototyped a solution that modified Spark core with ability to detect
scaling plug-ins.  This solution requires modifications to the spark upstream,
but does not provide any value over and above the proposed non-invasive
solution, and so it is no longer being considered.

## Affected Components

The primary affected component is the Oshinko system, which generates the
templates for Spark clusters.  It will need to be modified to declare the
necessary configurations for metrics and elastic daemon.

## Testing

Testing probably is going to be system/integration level.   It is relatively
easy to set up a Spark job that runs long enough to trigger increasing
Executor requests and then manually observe the number of Spark worker pods
as they increase and decrease.

The end-to-end testing system can be used to test elastic worker funtionality.

## Documentation

Any additional parameters to the template will need to be documented.
In my prototype I have parameters for the docker image for the elastic
python container.  Additional parameters could be settings for
SPARK_WORKER_CORES and SPARK_WORKER_MEMORY.  Whether or not to enable
elastic workers at all could also be made a parameter.   Other possible
parameters could be minimum workers and maximum workers.
