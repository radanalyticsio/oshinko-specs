# Revise the Implementation of Named Cluster Configs

## Introduction

Named cluster configurations are supported in oshinko
using configmap volumes. However, there is a simpler
implementation which gives the user more flexiblity and
eliminates a timing problem on update of the configmap.

## Problem statement

Currently the oshinko container will mount a configmap named
`oshinko-cluster-configs` as a volume at `/etc/oshinko-cluster-configs`
and use it as a source for named cluster configurations. If
a user creates a cluster referencing one of these named
configurations, oshinko will read the config and create
the cluster accordingly. The orgininal spec for this feature
is noted at the end of the document.

Configmaps mounted as volumes are a nice abstraction for
applications that are insulated from the fact that they
are running in kubernetes. However, oshinko is already
fully aware of kubernetes and in fact is behaving like
a controller for spark clusters so the insulating
nature of the volume abstraction is irrelevant.

Furthermore, there is a user workflow problem. A user
must edit the oshinko-cluster-configs configmap in
order to define new named configurations. When that
configmap is edited, there is a non-deterministic
delay before those changes are written through to
the oshinko pod. A user can guess and wait
"long enough", but without logging into the oshinko
pod to check the contents of /etc/oshinko-cluster-configs
or redeploying the oshinko pod to force a remount of
the configmap it is hard to be sure that the changes
have been written.

What we need is a system where new or modified named
cluster configurations are immediately available to
oshinko.

## Proposed solution

Instead of mounting a well-known configmap as a volume
which holds all known named cluster configs, oshinko
can simply use the kubernetes api to dynamically
retrieve a configmap on demand. When a cluster create
or update REST call is made, oshinko can GET
any named cluster configuration through the kube
client and read its list of key-value pairs. This
functionality replaces reading the same key/values
from the mountpoint.

Since the key value pairs are stored in a simple
map, processing of values is trivial. Furthermore,
timing problems are eliminated since oshinko will
always see the latest value for a new or newly
created configmap.

Default configurations will continue to work as
described in the original spec with one change.
In order to override the hardwired default, a
user will create a configmap named `oshinko-cluster-default`
(as opposed to defining a `default` entry in the
`oshinko-cluster-configs` configmap in the current
implementation).

### Alternatives

Leave the impl unchanged and search for a way to
deal with the edit/write-through issue with
oshinko.

## Affected Components

oshinko-rest

## Testing

Modify unit tests for the original impl

## Documentation

Documentation should be added explaining how the named
configurations work and how to specify them.

## References ##

[Pull request](https://github.com/radanalyticsio/oshinko-specs/pull/1) for the original spec
