# Add metrics reporting to Oshinko generated Spark clusters

## Introduction

Spark exposes several metrics about the state of the master and worker nodes
as well as information about jobs that are running or have run. These metrics
provide key data points that can be used for greater introspection into the
cluster state and health. This specification proposes adding a Graphite[[1]]
based metrics container to Oshinko generated Spark clusters with a service to
expose a RESTful endpoint for metrics queries.

## Problem statement

Spark has an internal metrics collection mechanism that can be used to
harvest an array of data points(see example 1). There are several available
"sinks" that can be used by Spark to expose the metrics data. By default,
Spark exposes a minimal REST endpoint that can be queried for metrics data,
this sink is referred to as the `MetricsServlet`[[2]]. Unfortunately, several
tests with this endpoint failed to produce useful results.

Although the `MetricsServlet` sink has proven light on details, the
`GraphiteSink` based on the Graphite project has proven quite valuable in
mining Spark for metrics. The primary difficulty when using this sink is
that it requires two other processes to be running alongside the Spark
master; a collection tool named Carbon, and the Graphite API server.

To effectively enable a Graphite based metrics solution, the pod spec that
Oshinko generates for Spark clusters will need to be changed to include the
two Graphite based contianers and a service will need to be created to allow
access to the metrics data. Images will also need to be generated for these
new processes, and the Spark images will need to have their configuration
files changed to allow metrics reporting.

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

To implement a Graphite metrics solution with the current Spark deployment
methodology, there will need to be changes to the oshinko-rest application,
the addition of a configuration file to current Spark images, and the
creation of two images for the Graphite and Carbon components.

The oshinko-rest application will need to be modified so that the deployments
for Spark master pods will include the new Graphite and Carbon images. An
example of what this looks like in YAML template form can be seen in
example 2. The deployment will also need a service created for the Graphite
API server to allow access to the metrics data.

An open question for the oshinko-rest implementation is whether the user
should be have the ability to disable the deployment of metrics, or if this
feature should always be deployed. For this specification, we should assume
that the metrics server is always deployed.

The Spark images currently in use will need to have a `metrics.properties`
file added to their configuration directory (`$SPARK_HOME/conf`). This file
is responsible for enabling the metrics and the configuring the sink in use.
The necessary components of that file can be seen in example 3.

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

### Alternatives

One alternative would be to explore the `MetricsServlet` code in the Spark
base to determine why it is not publishing data. If a root to this could be
found, then a patch could be created for the upstream Spark project to better
enable this sink. The main downside to this approach is the unknown factor
of community acceptance for such a change.

Another alternative would be to use one of the other sinks available in Spark.
Each of these sinks carries differing levels of complication. On the simple
end is the `CSVSink`, which writes metrics data to text files. This is
problematic due to the restriction of needing a volume to write and access
the files from, not to mention that the values themselves will need an
abstraction layer to expose. On the more complex end of the spectrum is the
`JmxSink`, which provides the metrics data to a JMX server which can display
the data in a console. The JMX option is more attractive in deployments that
are already using Java based infrastructures with support for managed beans.

## Affected Components

This change will initially affect only the oshinko-rest project. There is
potential in the future for exposing the metrics to the oshinko-webui, but
this is not included in this specification.

## Testing

The Graphite metrics server should be tested in the course of full end-to-end
functional testing. The server should be queried for presence and interaction
by retrieving metrics from a deployed cluster and ensuring that calls can be
made to specific metrics properties.

## Documentation

Documentation will be added to describe how a user might access the metrics.
This will include examples of how to make REST queries against the metrics
server with links to the upstream Graphite project where appropriate.

## References

1: https://graphite.readthedocs.io/en/latest/

2: https://spark.apache.org/docs/2.0.0/monitoring.html#metrics

[1]: https://graphite.readthedocs.io/en/latest/
[2]: https://spark.apache.org/docs/2.0.0/monitoring.html#metrics
