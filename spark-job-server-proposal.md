# Adding Spark Job Server To Run Spark Jobs

## Introduction

This document provides design of spark job server which would trigger spark
jobs either as scheduled or batch jobs. Providing the user a ui for inputing
the source code git location, environment variables required to integrate with
other apps, input parameters for spark job, selecting s2i builder image and
files required.

## Problem Statement

The Data Platform Spark applications can only run within OpenShift
environment and users are required to provide only source code which
currently we use source to image to create a standard image which will be run
against the spark cluster. Currently there is no way to schedule jobs or to run
cron jobs. Also if users have a more complex setup where they would need
JDG or other servers they would be required to do a lot of manual setup.

## Proposed Solution

Apache Spark users will submit their spark job in the form of binary jar or py
files. These files will require users to provide input as well as be triggered
by spark-submit shell script. However there are other environment requirements
that may be needed by users as well.

The requirements to create, run, rerun, stop a particular spark job by one
click of a button.

With a simple html form users can provide the following input:
* source git url
* input parameters for spark job
* files required to be mounted to run the job.
* inject environment variables from services that this spark job would need
to integrate with.
* specify if they would like a new cluster created or if they would like to use
an existing one
* schedule when the job should run
* how often the job should run

The workflow would be user fills out html form and oshinko-webui would send
json payload (see SampleJSON payload section below) with the information
required to build the s2i image and then run
a spark job including the inputs provided above. After that we would get an
update in the ui to see jobs running and their status. There would be a button
on the ui for users to rerun the spark job.

## SampleJSON Payload
For batch jobs:

```json
{
	clusterName: "sparkRecommendMLlib",
	gitUrl: "<url>",
	s2iImage: "docker.io/radanalyticsio/radanalytics-java-spark"
	sparkParams: "--class com.example.data.analytics.App /opt/recommend-mllib-1.0.0-SNAPSHOT-jar-with-dependencies.jar  --rank 5
	--numIterations 5 --lambda 1.0 --infinispanHost
	$RECOMMEND_SERVICE_SERVICE_HOST --kryo
	$SPARK_HOME/data/mllib/sample_movielens_data.txt",
	volumesMounts: ["vol-1"],
	servicesToLink: ["JDG"],
	runMode: "batch",
	runModeInput: "1"
}
```

Note: For batch mode you specify how many times it would run.

For scheduled jobs:

```json
{
	clusterName: "sparkRecommendMLlib",
	gitUrl: "<url>",
	s2iImage: "docker.io/radanalyticsio/radanalytics-java-spark"
	sparkParams: "--class com.example.data.analytics.App /opt/recommend-mllib-1.0.0-SNAPSHOT-jar-with-dependencies.jar  --rank 5
	--numIterations 5 --lambda 1.0 --infinispanHost
	$RECOMMEND_SERVICE_SERVICE_HOST --kryo
	$SPARK_HOME/data/mllib/sample_movielens_data.txt",
	volumesMounts: ["vol-1"],
	servicesToLink: ["JDG"],
	runMode: "scheduled",
	runModeInput: "*/5 * * * *"
}
```

Note: This would schedule the job to run every 5 minutes

## Alternatives

Alternative is to utilize templates to trigger a build which limits users to
only be able to create a single container with binary that runs the spark job.
Users would have to click add project and create the spark job everytime they
need to run.

## Affected Components

- oshinko-console : html form, angularjs javascript to make rest calls to
backend api that would create/list/start/stop spark jobs. No rest client
required as we will be leveraging existing openshift-api via openshift console
extension.

## Testing

Initially, the oshinko-rest code will be tested with unit tests to ensure that
the jobs are created and running. In cases of failure that the jobs get cleaned
up and the users is alerted that the job failed.

## Documentation

The documentation could provide examples of a word count example that could
be inputted by a user to see it in action. This will show users how easy it is
to run spark jobs on OpenShift.
