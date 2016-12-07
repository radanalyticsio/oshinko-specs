# ConfigMaps for Spark Configuration of the Driver Container

## Introduction

This specification outlines methods for exposing spark configuration
files through ConfigMaps in the container running the spark driver.

## Problem statement

Currently the only way to customize spark configuration files such
as log4j.properties and spark-defaults.conf for spark driver
applications is to create custom oshinko s2i images which contain
the custom settings. This is a very heavy-weight approach for
configuration, especially in situations where application
tuning is being done. Oshinko needs a more dynamic way to control
the cluster configuration files on spark drivers.

Spark driver containers are created from the OpenShift console
or CLI via templates. Consequently, the method of dynamic configuration
has to be something available from templates. Additionally, use
of a ConfigMap for custom spark driver configuration should be
optional -- the default behavior should be to use the configuration
defined in the image itself.

Note that this is a similar but different problem from providing
dynamic configuration of spark clusters (master and worker nodes).
The OpenShift objects which define a spark cluster are created
programmatically by oshinko, and so volumes and env vars can be
setup up by oshinko during  cluster creation based on values set
in JSON payloads to oshinko-rest.

## Proposed solution

ConfigMap objects mounted as volumes on the spark driver container
would provide a way to modify the default configuration for the
spark application.

ConfigMap objects can store multiple named configuration files and
be mounted to a particluar directory in a container. Consquently, it
is possible to specify log4j.properties, spark-defaults.conf, and other
files in a ConfigMap and mount them at a location such as `/opt/spark/oshinko-conf`
in the spark driver container. If the environment variable SPARK_CONF_DIR
is set to /opt/spark/oshinko-conf, then spark will read its configuration from
the ConfigMap.

The deploymentconfig that defines a spark application container can
specify that a ConfigMap be mounted at a well-defind location. However,
the fact that templates do not support conditional logic makes it more
difficult to support optional custom configuration -- a template either
always mounts a ConfigMap as a volume in a container, or it never does.
The template cannot decide.

The solution in this case is to always mount a well-known ConfigMap
in the spark driver containers. This ConfigMap will be created when
oshinko is launched and will initially be empty. It will be given
a name such as `oshinko-spark-driver-config` and will be mounted
at a well-known location like `/opt/spark/oshinko-conf`.

The oshinko application launcher will look in `/opt/spark/oshinko-conf`.
If the directory contains spark configuration files, then the launcher
will set SPARK_CONF_DIR to `/opt/spark/oshinko-conf`. If the directory
does not contain spark configuration files, SPARK_CONF_DIR will be
left alone and spark will run with the configuration set in the image.

The name of the ConfigMap to mount on the spark driver container will
be a parameter in the oshinko application templates.  It will have a
default value of `oshinko-spark-driver-config` and will be a required
value -- it must not be blank.

There will be 2 ways for a user to provide a custom configuration for
the driver:

* use OpenShift facilities to edit the ConfigMap `oshinko-spark-driver-config`.
  This will change the default spark driver configuration for any application
  launched in the project using the oshinko templates.

* create one or more ConfigMaps in the project to hold spark driver configurations.
  Set the parameter in the oshinko templates to the name of one of these ConfigMaps
  instead of `oshinko-spark-driver-config` when the application is launched.

A user-defind ConfigMap for a spark driver would look similar to this (shown in yaml):

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
      name: mysparkdriverconfig

The default `oshinko-spark-driver-config` ConfigMap would look like this,
with no data section:

    apiVersion: v1
    kind: ConfigMap
    metadata:
      creationTimestamp: null
      labels:
        app: oshinko
      name: oshinko-spark-driver-config
      
It is up to the user to create custom ConfigMaps with OpenShift
facilities before launching the application. If a specified ConfigMap
does not exist when the application is launched, an error will be returned
by OpenShift and the application will not run.

### Alternatives

There must be a method to dynamically specify the configuration
of the spark driver.  

One alternative mentioned here for completeness is to use clients
in the application pod to download a ConfigMap to the container
and write the contents to an EmptyDir() volume  mounted in the
driver container. This could be done in an init container in the
application pod.

The advantage here would be that it would not be necessary to create
an empty `oshinko-spark-driver-configs` ConfigMap as a default to be
used from templates. A template parameter would still exist that would
name a ConfigMap to use in the pod, but it would be blank by default.
A project that always ran with default configs on the spark driver
would never have an additional ConfigMap created.

The disadvantage is that it would be much more complex and really would
not make the application templates any simpler -- there would still
be an additional parameter on the screen. Also, updates to the
ConfigMap would not be written through to the pod automatically so
behavior would not be consistent with ConfMaps used in spark cluster
nodes. Additionally, it would require the driver pod to run using
the oshinko service account which it does not currently do.

## Affected Components

oshinko-s2i:

* The application launcher has to be modified to look in the
  well-known config directory and determine whether or not
  a non-empty ConfigMap has been mounted there. If so, it
  must set the SPARK_CONF_DIR env var appropriately

* The application templates must be modifed to specify the
  volumes and volume mounts in the pod and container specs.
  They must include a parameter which names the ConfigMap to
  mount as a volume with a default of `oshinko-spark-driver-config`

oshinko-rest:

* The `tool/server-ui-template.yaml` must be modifed to create the
  default spark driver ConfigMap when oshinko is launched. This is
  the same place where the `oshinko-cluster-configs` ConfigMap is
  created.

## Testing

Unit tests and end-to-end tests

## Documentation

Documentation would be needed in either case to describe how to
create and specify custom spark configuration for spark drivers.

