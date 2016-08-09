# Predefined Oshinko Environment Variables In Containers

## Introduction

Containers spun up under Oshinko will sometimes want access to the Oshinko
REST services that created them.  The needed information -- REST host/port,
Spark cluster name (if this is a spark cluster pod), possibly other information
-- can easily be provided as predefined environment variables that any
Oshinko-spawned container can assume to be defined.

## Problem statement

An example problem/use-case is the current "scorpion-stare" project to give
Spark clusters running under Oshinko the ability to add worker nodes to
themselves (or delete them) via the Oshinko REST interface.  The scale-out
plugins will need to know the REST server endpoint, and in this case also
what the Spark cluster's own "name" is with respect to oshinko's API.

## Proposed solution

Enhance the Oshinko REST server to define three new environment variables
for any container it starts up:

* OSHINKO_REST_HOST -- the host of the Oshinko REST server
* OSHINKO_REST_PORT -- the corresponding port for the server
* OSHINKO_SPARK_CLUSTER -- the name of this spark cluster WRT Oshinko's API.
This might only be defined if the container is a Spark master or worker node.

### Alternatives

I'm not aware of any reasonable alternatives, although Trevor mentioned that
it is possible to scrape the information from openshift programmatically.
That would be both inconvenient and probably not future-proof.

## Affected Components

The Oshinko REST server would set these environment variables for containers
at the time it spins them up.

## Testing

I'd propose some kind of integration test where containers are spun up with
a little application that prints out the values of these environment
variables, which would have to be captured from outside the containers by the
testing environment.

## Documentation

The documentation would be the names and semantics of each environment
variable.  Either in documentation of the REST server behavior, or in the
doc of invariant oshinko container properties, or both.
