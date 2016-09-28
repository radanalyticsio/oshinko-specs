# Cluster configs in the oshinko REST api

## Introduction

Currently the oshinko REST api allows Spark node counts
to be set when the cluster is created, but cluster
configurations will likely be more complex in the future.
This specification proposes that cluster configuration
settings be logically grouped in REST objects and that
named configurations be added as a feature.

## Problem statement

The initial oshinko REST api presents a very simple view of
cluster configuration: a user may specify the number of master
nodes and the number of worker nodes with fields in the REST
object (note however that at this time master count > 1 is meaningless).

As oshinko matures, cluster configurations may expand
to include things like HA master configuration, pod resource
limits, or other settings (disclaimer, not intended as a roadmap).
It makes sense to group cluster configuration settings logically
in the REST objects rather than include them as unrelated separate
fields.

Additionally, the ability to reference named configurations
as an alternative to explicit settings in a cluster object
would be a benefit to users in general and novice users in
particular. Given logical groupings, named configurations
become a possibility.

## Proposed solution

In the REST objects used to create and update clusters,
the master and worker count fields should be moved inside
of a `config` object along with a new `name` field.
In the future, the config object may contain additional
configuration fields. All fields become optional in this
new scheme but the default value of the name field will
be `default`.

Examples of cluster creation objects:

    {
      "name": "fred",
      "config":
              {
	        "masterCount": 1,
		"workerCount": 2
	      }
    }

    {
      "name": "skynet",
      "config":
              {
	        "name": "huge"
	      }
    }

    {
      "name": "slightlybigger",
      "config":
              {
	        "name": "small"
		"workerCount": 3
	      }
    }

The initial configuration for all clusters will
be the `default` configuration. If the config object
specifies the name of a stored configuration,
that stored configuration will be retrieved and used
to update the cluster configuration. Lastly, any
other settings in the config object will be used
to update the configuration again resulting in a
final set of values.

Named configurations can be managed through a ConfigMap
object, and that object can be mounted on the oshinko-rest
pod as a volume. This will provide persistent storage
for the named configurations within a project. Users
will be able to dynamically create and modify named
cluster configurations with this mechanism. In general,
named cluster configurations do not need to specify values
for all configuration setttings, with one exception.

There will be a hardcoded value for the `default`
cluster configuration in oshinko-rest which will always
be accessible. A user may override the default
configuration by adding a `default` to the ConfigMap
object. In this particular case, the user must specify
a value for every configuration setting or the new
default configuration will be considered invalid.

The oshinko-rest template will be modified to mount
a ConfigMap of a fixed name at a fixed location,
for example "oshinko-cluster-configurations" at
/etc/oshinko-cluster-configurations.

As a first pass, ConfigMaps will be managed using
the openshift CLI. In the future, we may want to
add an oshinko-rest endpoint for handling
cluster configurations in a more abstract way.

### Alternatives

None

## Affected Components

oshinko-rest
oshinko-web
oshinko-cli
oshinko-s2i (templates)

## Testing

Existing unit tests and end-to-end tests that deal with clusters
will be modified accordingly.

Additionally new unit tests should be added to test how
presence/absence of the namedConfig and config fields is
handled and the cluster configuration lookup mechanisms.

## Documentation

Documentation should be added explaining how the named
configurations work and how to specify them.