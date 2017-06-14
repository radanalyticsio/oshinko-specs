# Improve ephemeral cluster handling

## Introduction

This spec discusses ephemeral clusters in the context of Spark applications
built using oshinko S2I builders. Spark driver applications that are
constructed without the use of S2I are out of scope (although they could
conceivably use the same mechanisms as long as the extended CLI is
used to manage the cluster).

Oshinko has supported ephemeral clusters for a while but currently they only
truly work in a simple sunny-day, completed-application scenario. In the
sunny-day scenario the spark application completes and the driver deletes the
ephemeral cluster which it created.

If a driver pod is terminated (which can happen for various reasons discussed
below) before an application has completed, ephemeral clusters should still be
treated appropriately depending on the circumstances. Sometimes this will mean
deleting the cluster, and sometimes this will mean allowing the cluster to persist.

Note that in particular, long-running streaming applications may never complete
by design and therefore the intended method for stopping them is to delete the
deploymentconfig or job that launched the driver pod. In this case the current
handling will always leak ephemeral clusters.

## Problem statement

Ephemeral cluster management in Oshinko must handle all of the following
scenarios:

* A driver pod is interrupted before it completes

  This requires that the Oshinko `start.sh` script must handle termination
  signals. The signals that should be handled are SIGINT and SIGTERM.
  By default OpenShift will send a SIGTERM to a pod to shut it down, then
  wait for 30 seconds before sending SIGKILL if the pod is still running.
  SIGINT is less likely to be seen but could be sent to a container
  running `start.sh` by a user.

  Note that this general case covers these scenarios:
  * driver pod is scheduled to another node
  * driver pod replication controller is scaled to 0 (scale down or redeploy)
  
* A driver pod runs to completion

  If the pod was launched from a deploymentconfig, the pod will go to the
  'Completed' or 'Error' state depending on the exit code and then will enter
  'CrashLoopBackoff' before the deployment is restarted by OpenShift.

  If the pod was launched from a job, the pod will simply be marked
  'Complete' if the exit code indicates success, or will be restarted
  if the exit code indicates an error.

  In either case, it's possible that the deploymentconfig or job might
  be deleted while the pod is in Complete/Error/CrashLoopBackoff and a
  driver is not actively running. The solution must account for the fact
  that a dc or job may be deleted while no driver exists to receive a signal.

* A restarted driver pod uses a specified cluster name

  In cases where a user has set a specific cluster name on application launch,
  a restarted driver pod should use the same cluster name. It will be present
  in the pod's environment. Not that this can also apply to a *redeployed* spark
  driver since the cluster name is in the environment.

* A restarted driver pod uses a generated cluster name

  In cases where the user does not specify a particular cluster name and a
  random name is generated, a restarted driver pod needs a way to discover
  the generated name since it will not be present in the environment (it
  was not known at pod creation time). Note that this does not need to
  apply to a *redeployed* spark driver, but if a new generated name is used
  then of course the old cluster *must* be deleted.

* A driver pod should not be able to accidentally delete an ephemeral cluster
  associated with a different application, or a long-running cluster

* A driver pod must not be allowed to connect to a cluster in the process of
  being deleted.

* More than one application should not be allowed to use the same ephemeral cluster

  If deletion of an ephemeral cluster is tied to the lifetime of an application,
  then multiple applications cannot be allowed to share the same ephemeral cluster.
  That would make deletion semantics far too complicated and fragile.

  As explained below, as a first pass, multiple deployments of the same application
  will be treated as separate applications. That is, spark driver pods that are
  created by different replication controllers of the same deploymentconfig will
  not share an ephemeral cluster.  This could change in the future.

## Proposed solution

Note that in this spec we are only considering driver applications created using
deploymentconfigs. At this time we do not support ephemeral clusters created by
driver applications started from job objects. Making ephemeral clusters work
for job objects requires more work.

### Labeling to store metadata

Since oshinko is stateless and clusters are not represented in etcd as kubernetes
extensions, annotations and labels on OpenShift objects are really the only
"persistent storage" we have available for recording meta-data about associations
between clusters and applications.

This solution proposes adding the following labels on objects:

* `ephemeral`

  The value of this label is the name of the replication controller for
  the spark driver application, which is the name of the "deployment".
  This label goes on the spark master deploymentconfig. If this label is
  present, the spark cluster is ephemeral and is associated with the named
  deployment. For example:

      # bob-1 is the replication controller for deploymentconfig "bob"
      $ oc get rc
      NAME                 DESIRED   CURRENT   READY     AGE
      bob-1                1         1         1         8m
      cluster-ckdtw2-m-1   0         0         0         2s
      cluster-ckdtw2-w-1   0         0         0         2s

      # Partial export of the deploymentconfig for the spark master
      $ oc export dc cluster-4a6xhf-m
      apiVersion: v1
      kind: DeploymentConfig
      metadata:
        creationTimestamp: null
        generation: 1
        labels:
          ephemeral: bob-1
          oshinko-cluster: cluster-4a6xhf
          oshinko-type: master
        name: cluster-4a6xhf-m

* `uses-oshinko-cluster`

  The value of this label is the name of an oshinko cluster. This label
  goes on the replicationcontroller for the spark driver pod that creates
  the ephemeral cluster. For example:

      $ oc export rc bob-1
      apiVersion: v1
      kind: ReplicationController
      metadata:
        creationTimestamp: null
        generation: 1
        labels:
          app: oshinko-pyspark-dc
          application: oshinko-pyspark
          createdBy: template-oshinko-pyspark-dc
          openshift.io/deployment-config.name: bob
          uses-oshinko-cluster: cluster-ckdtw2
        name: bob-1

### Adding kubernetes downward api volume mounts

The spark driver deploymentconfigs will use features in the kubernetes downward
api to mount the label set from the dc on the driver pods as a volume.  This will
allow the `start.sh` script to look up relevant information stored in labels,
such as the name of the driver pod's deployment (replicationcontroller)

Knowing the deployment of the driver pod is essential for `start.sh` to manage
ephemeral clusters correctly. It allows the script to record a generated
cluster name for the deployment, create an ephemeral cluster associated with
the deployment, and guarantee that it only deletes an ephemeral cluster
associated with the deployment. This is accomplished by passing the name of
the deployment as the value of certain flags in oshinko CLI commands (more below).

### Extensions to oshinko CLI to enable ephemeral cluster features

To make ephemeral cluster handling work from within `start.sh` running
in a Spark driver pod, we will add options to the `create`, `delete`,
and `get` oshinko CLI commands:

* Options on create to mark the cluster ephemeral and associate the driver

      $ oshinko-cli create fred --app=<name of driver deployment> --ephemeral

  The value of the `--app` option may be either the name of the driver pod, or the
  name of the driver's deployment (both are available to `start.sh`). The `--ephemeral`
  option causes oshinko to mark the cluster as ephemeral (by default clusters are not
  ephemeral). The values of these options are used by oshinko to set the labels discussed
  above.

  Note that if the `--ephemeral` option is set, the `--app` option must also be set.
  However, the `--app` option may be used by itself to associate a driver with a
  long-running cluster.  Since the `uses-oshinko-cluster` label appears on the driver
  replicationcontroller, it's a many-to-one relationship and so multiple drivers can
  be associated with a long-running cluster.

* Options on delete to name the deployment and the termination status of the driver

      $ oshinko-cli delete fred --app=<name of driver deployment> --app-status=[completed|terminated]

  Both options must be present or both must be absent. If they are absent, then deletion of the
  cluster is unconditional, just like the historical `delete` command.

  In order for an ephemeral cluster to be deleted, the value of the `--app` option must match the
  value of the `ephemeral` label on the named cluster. Oshinko driver pods simply pass in the name
  of their own deployment.  This guarantees that they can only delete ephemeral clusters that
  they created. They cannot delete long-running clusters or ephemeral clusters created by other
  deployments.

  The `app-status` option indicates whether the spark driver application completed
  or was terminated early. In the case of S2I applications this has definite implications:

     * The `terminated` value for `app-status` indicates that the driver
       application was forcibly shutdown before `spark-submit` completed. This
       means the driver pod received a SIGTERM to shut it down while the spark
       code was still running (there are several scenarios where a pod may get this signal).

     * The `completed` value for `app-status` indicates that the driver application
       returned from the call to `spark-submit` and has issued a `delete` command.
       This is the last thing that `start.sh` does before exiting (or looping forever
       if APP_EXIT=false).

* Option on get to return clusters associated with a particular deployment

      $ oshinko-cli get --app=<name of driver deployment>
    
  This option on get causes oshinko to lookup the named deployment and search for the
  `uses-oshinko-cluster` label. If it's found, the named cluster is returned. The primary
  use in the S2I workflow is to allow a restarted driver pod to discover the name of the
  cluster it should use when cluster names are auto-generated. However, it can also be
  used as a handy way to find out what cluster a driver is connected to from the commandline.

* Ouput from get will include a column indicating if a cluster is ephemeral

  The value of this column will be the name of the driver deployment asociated with
  the ephemeral cluster. If the cluster is long-running, the value will be `<shared>`.
  See more below about explicit null values.

### Cluster deletion semantics

The rules for cluster deletion when `oshinko-cli delete` is called are:

* Any cluster may be deleted by anyone at anytime if the `delete` command is
  used with no options.

* If a delete command specifies the options described above, long-running clusters
  may never be deleted and an ephemeral cluster may only be deleted if the
  value of the `ephemeral` label matches the value of the `app` option.

* If labels and options match, an ephemeral cluster will always be deleted if
  the replica count of the driver deployment is 0. This means that the deployment
  has been scaled to 0 either as part of a deletion, redeploy, or a scale operation.
  It means that a driver that is part of the current deployment will not come back,
  or at least will not come back for some indeterminate amount of time.  If it does,
  it will make another cluster.

* If labels and options match, an ephemeral cluster will always be deleted if
  the driver replicationcontroller cannot be found. This means that the
  deployment has been deleted and is never coming back (note that a *different*
  deployment spawned by the same deploymentconfig *could* be created, but it's
  a different deployment so doesn't get to use this cluster).

*  If labels and options match and the replica count of the driver deployment
   is 1, an ephemeral cluster will be deleted if the value of `app-status` is
   `completed`. Since the application completed and the driver has not been scaled up
   (that is, there is no other driver pod using the cluster) the cluster serves
   no further purpose. The expectation is that application has run its course and
   will not be restarted.

   (In reality, because of the limitation of restart semantics for deploymentconfigs,
   the driver application *will* actually be restarted after a time if an operator
   does not delete the driver deploymentconfig or scale it to 0. However, since these
   things may happen after the driver pod has exited, the cluster must be deleted
   since the driver pod itself is the means of deletion. If a cluster is left and
   the driver deploymentconfig is deleted while in CrashLoopBackoff, the cluster will
   be leaked)

* Ephemeral clusters will always be left if the driver replica count is > 1.
  We don't yet have a use case for scaling a driver, but we can't stop it, and
  if multiple driver pods are using the same cluster then we can't allow one
  of them to delete the cluster while others may still be using it.

* An ephemeral cluster will be left if the replica count is 1 and the `app-status`
  is `terminated`. In this case we expect the driver to be restarted. The
  application was stopped early, and the most likely case is that a driver pod
  was rescheduled or killed purposely by an operator to case a restart. In this
  scenario it's much quicker to allow a new driver pod to use the same cluster.

  If the intention was to halt the application entirely, then the driver deploymentconfig
  would have been deleted or the deployment would have been scaled to 0.

### Application startup semantics in `start.sh`

There are a few rules governing startup of a driver pod:

* If a driver pod has a cluster name specified in the environment, it just
  uses that name for the cluster. If it does not have a name specified, it
  looks to see (via oshinko-cli) if there is a name recorded in a label on
  its replicationcontroller.

* If a driver pod looks up a cluster name to see if it's in use and finds a cluster
  with `Incomplete` status, it waits up to a minute for the cluster to
  disappear or have its status changed to `Running`. This accounts for clusters
  in the process of being deleted or created. If an incomplete cluster does not
  resolve in 1 minute, the driver pod exits.

* If a driver pod looks up a cluster name and finds an ephemeral cluster, it
  compares its own deployment value to the value of the `ephemeral` label on
  the found cluster (as returned from oshinko-cli). If the two deployments
  don't match, the driver pod exits.

* If oshinko cannot find the deployment for the driver pod, it will create a
  long-running cluster even if an ephemeral cluster was asked for. This covers
  the case where a driver is started using kubernetes jobs or some other mechanism.

### Explicit null values in ouput from CLI `get`

Since `start.sh` parses the output of oshinko-cli, it's necessary for all
columns printed by `get` to have an explicit value (except for potentially the
last column). The bash script cannot be expected to figure out which fields are
present and which are not.

Consequently, null values should be added for certain columns:

* <missing> will be used for services and routes that might not exist
* <shared> will be used for long-running clusters that have no associated deployment

## Alternatives / Additional (in no particular order)

### Make ephemeral cluster logic with with jobs

Jobs don't use replicationcontrollers, so the mechanisms we use to
track ephemeral clusters and their associated deployments will need to be
redesigned in the case of jobs. We also will need a scheme to handle
parallelism, number of completions, and potentially other aspects of jobs.

### Consider the exit status of spark-submit

There might be some consideration of the exit status of spark-submit,
especially when we extend this solution to jobs. If a spark application
does not complete with success, we can imagine that it may be restarted
even if technically the pod ran to completion. In that case we may want
to allow an ephemeral cluster to remain

### Add a CLI command to associate a driver with a long-running cluster

Currently we only do association on create. However, the label mechanism
will work to associate a driver with a long-running cluster, so we may
want to add a CLI command to handle association/disassociation of
drivers with long-runnign clusters.

### Add a CLI command to lookup all the drivers using a shared cluster

Since shared (long-running) clusters may be used by multiple drivers, given
the above we may want to add a command to lookup all the drivers using
a shared cluster. Or maybe it just prints as part of `get`.

### Allow a driver to use more than one cluster

Association currently is just a simple key/value pair. Not sure it makes
sense for a driver to be associated with more than one cluster, but maybe.
In that case it needs to be a list.

### Allow clusters to be shared by multiple deployments from the same dc

This really comes down to allowing an ephemeral cluster to persist across
a redeploy of the driver. This is possible, but oshinko needs logic to
look at the current state of the deploymentconfig and determine if it
is spinning up a successor replicationcontroller while spinning down the
current one.  If we can figure that out, this is possible.

It begs the question of whether or not we always want to recreate clusters
if we redeploy the driver, and this may be informed by a deeper look into
how we do rolling upgrades (investigation coming soon).

So maybe we don't want to pursue this yet ...

### Move CLI extensions for ephemeral clusters to normal build

The extra CLI command options are currently all in the `make extended`
version of oshinko-cli and are only used in the S2I pods. We could consider
moving those options to the normal CLI.

However, we don't want to confuse users. If users are creating "ephemeral"
clusters outside of an S2I workflow, maybe the user app that does the creating
should be tracking clusters itself ...

### Add a watcher container to the spark master pod to check for abandoned clusters

This would be a simple container on the spark master pod that simply polls
the oshinko-cli using some not-yet-invented command to check if the cluster
is abandoned, i.e. it's ephemeral and the driver deployment cannot be located.

The extra container could be added only to ephemeral clusters, and it could
poll slowly (once every 5 seconds).

If it found itself to be abandoned, it could call oshinko-cli delete on itself.
This should work, because kubernetes sends a SIGTERM with a 30 second delay
on a delete.  The call should complete.

There may not be known edge cases where we end up with abandoned clusters
at the moment, but this would guarantee cleanup if such edge cases or
errors exist.

## Affected Components

oshinko-core
oshinko-cli
oshinko-s2i
oshino-web (as far as expected values from `get`)
oshinko-rest (as far as changed parameters to oshinko-core call)

## Testing

An entire automated test suite built on the OpenShift CLI test suite
tools will be developed to test ephemeral cluster handling in all
scenarios. It will be submitted with the finished work.

## Documentation

Any existing docs dealing with ephemeral clusters will need to be
updated and expanded. If no such doc exists at present, one will
need to be added.

