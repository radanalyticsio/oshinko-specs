# ConfigMaps for Spark Configuration Files

## Introduction

This specification outlines methods for exposing Spark configuration
files through ConfigMaps.

## Problem statement

Currently the only way to customize spark configuration files such
as log4j.properties and spark-defaults.conf for clusters
launched from oshinko is to create a custom spark image and redeploy
oshinko to use that image. This is a very heavy-weight approach for
configuration, especially in situations where cluster or application
tuning are being done. Oshinko needs a more dynamic way to control
the cluster configuration files.

Additionally, since a spark image is shared between master and workers,
there is no way using the image method to differentiate configs between
master and worker nodes. One size must fit all.

## Proposed solution

ConfigMap objects mounted as volumes on the spark master and worker
nodes would provide a way to modify the default configuration for
a cluster without creating a new image.

ConfigMap objects can store multiple named configuration files and
be mounted to a particluar directory in a container. Consquently, it
is possible to specify log4j.properties, spark-defaults.conf, and other
files in a ConfigMap and mount them at a location such as /opt/spark/oshinko-conf
in a spark node.  If the environment variable SPARK_CONF_DIR is set
to /opt/spark/oshinko-conf, then spark will read its configuration from
the ConfigMap.

There are two approaches to using ConfigMaps for spark configuration
in oshinko.  The simpler approach will be outlined here, and the more
complex approach briefly described under `Alternatives`. Since this is
an important feature related to cluster/application tuning and performance
metrics, this spec recommends pursuing the simple approach first which can
be done in a more timely manner and then perhaps implementing the alternative
at a later time.

In the simple approach, `sparkMasterConfig` and `sparkWorkerConfig` fields
are added to the cluster config JSON object. The values of these fields name
ConfigMaps to use as a source for spark configuration files on the master
and worker nodes respectively:

    {
      "name": "fred",
      "config":
              {
                "masterCount": 1,
                "workerCount": 2,
                "sparkMasterConfig": "mymasterconfig",
		"sparkWorkerConfig": "myworkerconfig"
              }
    }

In this example, `mymasterconfig` and `myworkerconfig` are the
names of ConfigMaps which contain spark configuration settings that
would look similar to this (shown in yaml):

    apiVersion: v1
    data:
      log4j.properties: |-
        log4j.rootCategory=INFO, console
        log4j.appender.console=org.apache.log4j.ConsoleAppender
        log4j.appender.console.target=System.err
        log4j.appender.console.layout=org.apache.log4j.PatternLayout
        log4j.appender.console.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n
      spark-defaults.conf: |-
        spark.executor.memory
    kind: ConfigMap
    metadata:
      creationTimestamp: null
      labels:
        app: oshinko
      name: myworkerconfig

Note that `sparkMasterConfig` and `sparkWorkerConfig` could be set to
the same value if all nodes in the cluster have the same configuration.
Additionally, both are optional -- it is possible to customize the
configuration for the workers and leave the master unmodified or
vice versa.

If oshinko sees either of these fields set in the cluster config
object, it will mount the named ConfigMap as a volume at a fixed
location on spark containers of the appropriate type and set
the SPARK_CONF_DIR environment variable appropriately.

It is up to the user to independently create the named ConfigMaps
with OpenShift facilities before launching the cluster. If a
specified ConfigMap does not exist when oshinko creates the cluster,
an error will be returned and the cluster will not be created. By
default OpenShift will not create a container until all referenced
ConfigMaps exist (the container will be pending).

Note that clusters which do not specify these ConfigMaps will have the
static spark configuration which was set in the spark image being
used.

### Alternatives

Oshinko already supports named configurations for cluster configs.
An alternative approach is to allow the contents of the configuration
files to be specified directly as part of a named configuration. For
example, here are the contents of the `oshinko-cluster-configs` ConfigMap
where the `small` configuration contains a log4j.properties file for
spark worker nodes(shown in yaml):

    apiVersion: v1
    data:
      large.workercount: "10"
      medium.workercount: "5"
      small.worker.log4j.properties: |-
        log4j.rootCategory=INFO, console
        log4j.appender.console=org.apache.log4j.ConsoleAppender
        log4j.appender.console.target=System.err
        log4j.appender.console.layout=org.apache.log4j.PatternLayout
        log4j.appender.console.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n
      small.workercount: "3"
    kind: ConfigMap
    metadata:
      creationTimestamp: null
      labels:
        app: oshinko
      name: oshinko-cluster-configs

In this approach, a list of supported configuration files would be added to
Oshinko. If a cluster was created using a named configuration and that
named configuration specified known files for master or worker nodes,
the contents of those files would be added to a new ConfigMap generated
for nodes of the given type. The cluster nodes would be created with those
ConfigMaps mounted at a known location and with the SPARK_CONF_DIR env var set
appropriately.

The advantage of this approach is that a user would not have to independently
create additional ConfigMaps for alternate spark configurations. They would
simply have to edit the existing `oshinko-cluster-configs` ConfigMap where
named configurations are already created and bundle the spark configuration
files with the rest of the cluster settings.

This approach is more difficult because it requires Oshinko to create
ConfgMap objects dynamically and to associate them with the cluster so that
they may be cleaned up appropriately when the cluster is destroyed.

## Affected Components

For either approach, oshinko-rest.

For the simple approach, oshinko console to the extent that it exposes
forms to allow setting values in a cluster config object on create and
would need to allow the `sparkConfig` setting. In the second approach,
oshinko console would not have to support anything other than named configs
(which should be supported anyway)

## Testing

Unit tests and end-to-end tests

## Documentation

Documentation would be needed in either case to describe how to
add the custom spark configurations. It would be similar in either
case.

