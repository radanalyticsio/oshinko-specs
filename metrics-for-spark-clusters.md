# Add metrics reporting to Oshinko generated Spark clusters

## Introduction

Spark exposes several metrics about the state of the master and worker nodes
as well as information about jobs that are running or have run. These metrics
provide key data points that can be used for greater introspection into the
cluster state and health. This specification proposes adding a metrics
deployment option to Oshinko generated Spark clusters, with the ability
to expose a RESTful endpoint for metrics queries.

## Problem statement

Spark has an internal metrics collection mechanism that can be used to
harvest an array of data points(see example 1). There are several available
"sinks" that can be used by Spark to expose the metrics data. By default,
Spark exposes a minimal REST endpoint that can be queried for metrics data,
this sink is referred to as the `MetricsServlet`[[1]]. Although the
`MetricsServlet` can be used to harvest some metrics data, it does not
provide a rich interface to the metrics.

To promote deeper introspection on running Spark clusters, the oshinko
deployment methodology should be modified to include support for one of the
richer metrics interfaces provided by the `JmxSink` or `GraphiteSink`
implementations. These implementations provide a queryable interface to the
metrics data provided by Spark. This interface will allow applications that
wish to consume the metrics a much finer degree of control over the data
they inspect.

**Example 1, sample metrics topics**

```
"jvm.PS-MarkSweep.count", 
"jvm.PS-MarkSweep.time", 
"jvm.PS-Scavenge.count", 
"jvm.PS-Scavenge.time", 
"jvm.heap.committed", 
"jvm.heap.init", 
"jvm.heap.max", 
"jvm.heap.usage", 
"jvm.heap.used", 
"jvm.non-heap.committed", 
"jvm.non-heap.init", 
"jvm.non-heap.max", 
"jvm.non-heap.usage", 
"jvm.non-heap.used", 
"jvm.pools.Code-Cache.committed", 
"jvm.pools.Code-Cache.init", 
"jvm.pools.Code-Cache.max", 
"jvm.pools.Code-Cache.usage", 
"jvm.pools.Code-Cache.used", 
"jvm.pools.Compressed-Class-Space.committed", 
"jvm.pools.Compressed-Class-Space.init", 
"jvm.pools.Compressed-Class-Space.max", 
"jvm.pools.Compressed-Class-Space.usage", 
"jvm.pools.Compressed-Class-Space.used", 
"jvm.pools.Metaspace.committed", 
"jvm.pools.Metaspace.init", 
"jvm.pools.Metaspace.max", 
"jvm.pools.Metaspace.usage", 
"jvm.pools.Metaspace.used", 
"jvm.pools.PS-Eden-Space.committed", 
"jvm.pools.PS-Eden-Space.init", 
"jvm.pools.PS-Eden-Space.max", 
"jvm.pools.PS-Eden-Space.usage", 
"jvm.pools.PS-Eden-Space.used", 
"jvm.pools.PS-Old-Gen.committed", 
"jvm.pools.PS-Old-Gen.init", 
"jvm.pools.PS-Old-Gen.max", 
"jvm.pools.PS-Old-Gen.usage", 
"jvm.pools.PS-Old-Gen.used", 
"jvm.pools.PS-Survivor-Space.committed", 
"jvm.pools.PS-Survivor-Space.init", 
"jvm.pools.PS-Survivor-Space.max", 
"jvm.pools.PS-Survivor-Space.usage", 
"jvm.pools.PS-Survivor-Space.used", 
"jvm.total.committed", 
"jvm.total.init", 
"jvm.total.max", 
"jvm.total.used", 
"master.aliveWorkers", 
"master.apps", 
"master.waitingApps", 
"master.workers"
```

## Proposed solution

After several technology reviews, the proposed solution to this problem is
to implement a `JmxSink` based metrics deployment using the Jolokia[[2]]
project. Jolokia provides several options for deployment, as well as a
RESTful server for interacting with the metrics data.

JMX is a protocol for the Java Management Extensions, which allow specific
code portions to hook into the Java Virtual Machine with the end goal of
allowing management behaviors. Using the JMX path to expose metrics will give
the deployed clusters a high degree of flexibility with regards to integrating
into the wider logging effort in OpenShift.

There is a question as to the best deployment path with Jolokia though. The
two primary methods that oshinko could use for adding Jolokia are as a JVM
agent attached to the master node in a Spark cluster, or as a proxy agent
in a container with network access to the JVM in a master node.

### Jolokia as a JVM agent

The simplest route to deploying Jolokia is to use the agent path. This change
would require adding the Jolokia agent JAR file to the OpenShift Spark
image and then changing the launch parameters for master deployment. In
effect this would look as follows:

```
spark-class -javaagent:jolokia-agent.jar=port=19150,host=spark-master-123 org.apache.spark.deploy.master.Master
```

In this example, the Jolokia REST server is started on port `19150`, with the
hostname being `spark-master-123` (this value is for example purposes only,
in production the service name would be used). When launched, this pod could
be queried for metrics data. If the metrics were needed outside of the pod,
a service would need to be exposed allowing this acquisition.

### Jolokia in proxy mode

A more complicated method for adding Jolokia is to deploy it using the
remote proxy methodology. In this form, a new container is created that
contains a Tomcat HTTP server with the Jolokia agent. Requests that are
made to the Jolokia agent through the HTTP server would have a clause
directing them to the exposed JMX port on the JVM they wish to inspect.

To properly enable the proxy mode, the launcher for the Spark master
deployment would need to be changed to include a few specific definitions for
the JVM. It would look something like this:

```
spark-class -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=19150 -Dcom.sun.management.jmxremote.local.only=false -Djava.rmi.server.hostname=127.0.0.1 -Dcom.sun.management.jmxremote.rmi.port=19151 org.apache.spark.deploy.master.Master
```

This would start the JVM for the master deployment with the JMX remote
connection available on port `19150` for all local connections. In this manner
only the local container with the HTTP server would be allowed to connect to
the remote port.

For this deployment, the pod spec for the Spark master would be changed to add
the new container with the HTTP server and Jolokia agent.

### Comparison

Using Jolokia in agent mode attached directly to a Spark master is the most
straightforward approach possible. This would require the addition of a
Jolokia JAR file to the Spark master images. It would also require a small
modification to the launcher script. This deployment would also then cause
the master node to serve HTTP content from the Jolokia agent. It is unclear
whether there will be performance impacts from running this style of
deployment.

In proxy mode, the deployment is more complicated as a new container image
will need to be created and maintained by the oshinko team. The main benefit
of this deployment is that the service for providing metrics can be scaled
independently from the actual Spark node. There may be latency impacts based
on having an extra REST server in the pipeline for aquiring the metrics data.

Both implementations appear to provide the same data, with the same ability
to craft specific requests to the Jolokia REST server. The primary
differentiator is the separation of the REST server into a container of its
own.

### General changes

Regardless of the methdology chosen, the `metrics.properties` configuration
file will need to be adjusted to include the following lines:

```
*.sink.jmx.class=org.apache.spark.metrics.sink.JmxSink

master.source.jvm.class=org.apache.spark.metrics.source.JvmSource
worker.source.jvm.class=org.apache.spark.metrics.source.JvmSource
driver.source.jvm.class=org.apache.spark.metrics.source.JvmSource
executor.source.jvm.class=org.apache.spark.metrics.source.JvmSource
master.source.executors.class=org.apache.spark.metrics.source.Source
```

### Alternatives

The primary alternative to the JMX based implementation is to use the
Graphite base sink. The following text describes how a Graphite solution
would be implemented. The primary downsides to using Graphite are that it
does not provide an easy pathway to integrate with the Hawkular based
metrics that OpenShift uses, and that the storage for individual metrics
data is contained within the containers created for the Graphite and Carbon
serivces.

To implement a Graphite metrics solution with the current Spark deployment
methodology, there will need to be changes to the way that oshinko based
applications deploy the Spark clusters. The addition of a configuration file
to current Spark images, and the creation of two images for the Graphite and
Carbon components will be required.

Oshinko based applications will need to be modified so that the deployments
for Spark master pods will include the new Graphite and Carbon images. An
example of what this looks like in YAML template form can be seen in
example 2. The deployment will also need a service created for the Graphite
API server to allow access to the metrics data.

Images will need to be created for the Graphite API server and the Carbon
storage application. These images should be based on Centos for public
solutions and hosted on the Docker registry alongside current images. In this
manner, these images can be used by any Oshinko deployment. As these images
may require customization beyond a simple docker file, directories should be
created in the oshinko-rest project under the top-level `tools` directory to
house them.

It is worth noting that one of the primary strengths of a Graphite-based
solution is the ease of access to the underlying metrics data. The Graphite
API server exposes a RESTful endpoint for querying the data, this interface
allows retrieval of the data in several structured formats (eg. JSON) as well
as graphical representations (eg. SVG).

**Example 2, DeploymentConfig for Spark master with Graphite**

```
- kind: DeploymentConfig
  apiVersion: v1
  metadata:
    name: ${MASTER_NAME}
    labels:
      oshinko-cluster: ${CLUSTER_NAME}
      oshinko-type: master
  spec:
    strategy:
      type: Rolling
    triggers:
      - type: ConfigChange
    replicas: 1
    selector:
      name: ${MASTER_NAME}
    template:
      metadata:
        labels:
          name: ${MASTER_NAME}
          oshinko-cluster: ${CLUSTER_NAME}
          oshinko-type: master
      spec:
        volumes:
          - name: whisper
            emptyDir:
              medium: ""
        containers:
          - name: ${MASTER_NAME}
            image: ${SPARK_MASTER_IMAGE}
            env:
              - name: SPARK_MASTER_PORT
                value: "7077"
              - name: SPARK_MASTER_WEBUI_PORT
                value: "8080"
            ports:
              - containerPort: 7077
                protocol: TCP
              - containerPort: 8080
                protocol: TCP
          - name: ${MASTER_NAME}-graphite
            image: ${GRAPHITE_IMAGE}
            ports:
              - containerPort: 8000
                protocol: TCP
            volumeMounts:
              - name: whisper
                mountPath: /var/lib/carbon/whisper
          - name: ${MASTER_NAME}-carbon
            image: ${CARBON_IMAGE}
            ports:
              - containerPort: 2003
                protocol: TCP
            volumeMounts:
              - name: whisper
                mountPath: /var/lib/carbon/whisper
```

**Example 3, metrics.properties important bits**

```
*.sink.graphite.class=org.apache.spark.metrics.sink.GraphiteSink
*.sink.graphite.host=127.0.0.1
*.sink.graphite.port=2003
*.sink.graphite.period=10
*.sink.graphite.unit=seconds

master.source.jvm.class=org.apache.spark.metrics.source.JvmSource
worker.source.jvm.class=org.apache.spark.metrics.source.JvmSource
driver.source.jvm.class=org.apache.spark.metrics.source.JvmSource
executor.source.jvm.class=org.apache.spark.metrics.source.JvmSource
master.source.executors.class=org.apache.spark.metrics.source.Source
```

## Affected Components

The openshift-spark project will need to be changed to include changes for
the launcher script, regardless of the methodology chosen.

The oshinko-core project will also need to be changed depending on the type
of deployment required.

## Testing

Testing will be formed by deploying clusters and then aquiring metrics data
from the exposed REST server. This should confirm whether the server is
working and relaying good information.

## Documentation

Documentation will be added to describe how a user might access the metrics.
This will include examples of how to make REST queries against the metrics
server with links to the upstream Graphite project where appropriate.

## References

1: https://spark.apache.org/docs/2.0.0/monitoring.html#metrics

[1]: https://spark.apache.org/docs/2.0.0/monitoring.html#metrics
