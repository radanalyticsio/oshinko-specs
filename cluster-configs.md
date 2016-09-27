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

The master and worker count fields in any current REST objects
should be replaced with a configuration object that contains
both fields and may in the future contain other fields. For
example, a JSON cluster creation object would like like this:

    {
      "name": "fred",
      "config":
              {
	        "masterCount": 1,
		"workerCount": 2
	      }
    }
			   
A `namedConfig` field should be added that
can be used to refer to a stored configuration by name.
For example:

    {
      "name": "fred",
      "namedConfig": "small",
    }

If the namedConfig field is set, the value will be
used to look up a stored configuration. If the
configuration is not found an error will be returned,
otherwise the configuration will be read and used
as the initial configuration for the cluster. If
the `config` field is also set, any fields in the
config object will be used to update the fields
read from the named configuration. In this way
a user can tweak an existing named configuration
if desired without creating a new, separate named
configuration or specifying an entire configuration
in the config object if something similar exists.

Named configurations can be managed as a ConfigMap
object, and that object can be mounted on the oshinko-rest
pod as a volume. This will provide persistent storage
for the named configurations within a project. Users
will be able to dynamically create and modify named
cluster configurations with this mechanism.

The oshinko-rest template can be modified to mount
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

There may be no existing documentation that is affected aside
from the swagger generated API documentation.