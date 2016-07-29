# Add container health probes to Spark clusters

## Introduction

This document is about adding container probes to the Spark cluster pods that
are spawned by the Oshinko REST server.

## Problem statement

Currently, a deployed Spark pod's health is not monitored by Kubernetes or
OpenShift. The replication controllers created during deployment are the only
tools for automatically restarting containers that have crashed or failed
somehow. In circumstances where a Spark container becomes unresponsive but
does not crash, there is no mechanism implemented to restart those containers.

Furthermore, when services are deployed against the Spark master pod for its
web and application interfaces, there is nothing to prevent connections from
being created before the pod is ready. This can lead to a condition where a
pod may attempt a connection, and traffic, before the pod is available.

## Proposed solution

Kubernetes provides
[container probes](http://kubernetes.io/docs/user-guide/pod-states/#container-probes)
as a mechanism for automatically controlling the restart behavior and network
access to pods. There are two types of probes provided; liveness and
readiness.

Liveness probes are a simple recurring detection event(there are several
styles to choose from) which will invoke a pod's restart policy after a
configurable number of probe failures. Through this mechanism, Kubernetes will
periodically determine a pod's health, and react appropriately under failure
conditions.

Similarly, a readiness probe is a recurring detection event that will inform
Kubernetes that a pod with a service is ready to receive traffic. When a pod
fails a readiness check, its IP address is removed from routing tables that
are associated with its service. In this manner, Kubernetes can provide a more
thorough network model, ensuring that a pod will not be connected to a service
until that service is ready for traffic.

All nodes in a Spark cluster supply some sort of web interface, the master
exposes a control panel style application, whereas the workers supply
informational style pages. These pages can be used as a health check source
for the `HTTPGet` style of liveness probe. This would permit Kubernetes to
restart any pod that enters a state where it has not exited but no longer is
responding to HTTP requests.

Likewise, the Spark master node exposes its web address as a routeable
endpoint through a service. This endpoint could be used as the source for a
readiness check on those master nodes.

Given the simplicity of these probes and the fact that Spark currently has
exposed HTTP endpoints, adding liveness and readiness checks will require
changing the container specs for the master and worker pods.

### Alternatives

One alternative to this approach would be to implement our own custom
in-container processes for responding to liveness and readiness probes. This
may provide a higher degree of accuracy as a custom probe responder could
provide a more in-depth system analysis than simply checking for the presence
of the HTTP ports. This option is not precluded by the work proposed in this
spec, a custom probe responder could be added later using the same
mechanisms as proposed.

Along the lines of custom probe responders would be an option of using a
method beyond the HTTP GET for the check. Kubernetes provides other methods
for checking a socket or running a custom shell command in the targetted
container. One of these options could be used to increase the accuracy of
the probe check. This option would require creating the custom applications
which would live in the Spark containers for the probes.

Another alternative would be to not use the Kubernetes probes and instead
allow the Oshinko REST server to periodically check to determine the state
of Spark cluster pods. This option could become quite involved as it would
require the Oshinko server to implement its own monitoring framework.

Doing nothing is also an option. As it currently stands, Kubernetes will
restart any Spark containers that exit, and thus end. This would cover
situations where there is an abnormal stoppage of the Spark applications, it
would not cover circumstances where the Spark process fails but does not exit.
This also does not address the readiness probe case.

## Affected Components

* oshinko-rest will need to have its routines for creating the Spark cluster
  objects updated to include the probe objects.

* Spark image should be updated to add the probes into the OpenShift template
  included in the repository.

* openshift-spark should also have its template updated.

## Testing

Initially, the oshinko-rest code will be tested with unit tests to ensure that
the proper probe objects are added to its clusters. As more thorough
functional tests are added, custom tests should be created to ensure that
the negative cases are exercised(ie introduce failing probe checks).

## Documentation

This change should not require large documentation, but should make it clear
to end users how the probes are working.
