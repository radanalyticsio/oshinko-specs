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
			   
Additionally, a `stored-config` field should be added that
can be used to refer to a stored configuration by name. If
the stored-config field is set, the config object will be
retrieved and those settings will be used.  For example:

    {
      "name": "fred",
      "stored-config": "small",
    }

The `config` and `stored-config` fields will be optional
and mutually exclusive. That is, one and only one must
appear. If both or neither are present, an error will
be returned.

If a `stored-config` field refers to a config object which
does not exist an erorr will be returned.

As a first pass, oshinko will include a set of fixed
configurations that may be used to create clusters of
different sizes (`small`, `medium`, and `large`).

In a future spec, options for persistent storage of cluster
configurations and definition/modification of stored configs
by end users will be explored.

### Alternatives

Do not support stored configs. However, logical grouping
of configuration settings is still desirable.

## Affected Components

oshinko-rest
oshinko-web
oshinko-cli

## Testing

Existing unit tests and end-to-end tests that deal with clusters
will be modified accordingly. Additionally, unit tests should be
added to verify that the new fields are optional and mutually
exclusive.

## Documentation

There may be no existing documentation that is affected aside
from the swagger generated API documentation.