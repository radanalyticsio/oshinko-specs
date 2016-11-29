# Adding Spark Job Server To Run Spark Jobs

## Introduction

This document provides design of spark job server which would trigger spark
jobs either as scheduled or  batch jobs. Providing the user a ui for inputing
the jar location, environment variables required to integrate with other apps,
input parameters for spark job and files required.

## Problem Statement

Spark Applications can only run within OpenShift environment and users are
required to do provide only source code which currently we use source to image
to create a standard image which will be run against the spark cluster.
Currently there is no way to schedule jobs or to run cron jobs. Also if
users have a more complex setup where they would need infinispan-server or
other servers they would be required to do a lot of setup.

## Proposed Solution

Apache Spark users will submit their spark job in the form of binary jar or py
files. These files will require users to provide input as well as be triggered
by spark-submit shell script. However there are other environment
requirements that may be needed by users as well.

The requirements to create, run, rerun, stop a particular spark job by one
click of a button.

With a simple html form users can provide the following input:
* jar location / source git url
* input parameters for spark job
* files required to be mounted to run the job.
* inject environment variables from services that this spark job would need
to integrate with.

## Alternatives

Alternative is to utilize the kube api since oshinko is currently making rest
calls and ask the kubeapi to create a kubernetes job that will run the spark
job for us.

## Affected Components

- oshinko-rest : Will need to have server-side rest api for
create/list/start/restart/delete Spark job within kubernetes and have error
handling incase of failure scenario. Also the vserver-side code would need to
integrate with kubeapi.
- oshinko-web : html form, angularjs javascript to make rest calls to backend
api that would create/list/start/restart/delete spark jobs.

## Testing

Initially, the oshinko-rest code will be tested with unit tests to ensure that
the jobs are created and running. In cases of failure that the jobs get cleaned
up and the users is alerted that the job failed.

## Documentation

The documentation could provide examples of a word count example that could
be inputted by a user to see it in action. This will show users how easy it is
to run spark jobs on OpenShift.
