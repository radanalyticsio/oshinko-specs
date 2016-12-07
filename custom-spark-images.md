# Custom Spark Images

## Introduction

Often, for more advanced Spark users, there is a need to define extra
modules and configurations that should be present in the individual
containers of a Spark cluster. This specification defines a path
for users to inject their own Spark images into the deployment pipeline.

## Problem statement

Currently, there are limited functions for redefining the Spark image that
is deployed by the Oshinko suite of applications. For example, the
oshinko-rest server can accept an environment variable which defines the
Spark image it should use for deployment. But, this is only on a per-project,
per-installation basis.

Additionally, even when the image is changed during Oshinko installation,
this image is fixed for all subsequent deployments of Spark in that project.

These configurations restrain a user from being able to modify the Spark
image on a per-application basis. Further, they require reinstallation of
the Oshinko pieces or configurations for the new changes to take effect.

Redefining the Spark image that Oshinko will use for deployment is
considered to be a feature for advanced users and those who are testing
deployments. It is not to be considered as a first line for new users.

## Proposed solution

To address the issue of custom Spark images, the oshinko-rest server should
be changed to accept an image URI in its configuration section during
REST calls to create new clusters. This parameter will be optional, with the
default behavior being to use the image that was designated during install.

The oshinko-console extension should be modified to optionally accept an
image URI during display of the new cluster modal dialog. This option should
be listed in a closed fold in the modal dialog for advanced features so as
not to confuse new users on the requirements for deploying a Spark cluster.

*Example 1, proposed REST for cluster creation*
```
{
    "config": {
        "imageUrl": "docker.io/myrepo/openshift-spark:custom"
    },
    "name":"custom"
}
```

### Alternatives

A choice that needs to be considered is whether or not to allow this
functionality at all. By allowing users to define their own Spark images for
deployment, we will be opening a doorway that could create other issues. For
example, a user could create a custom image that would not be suitable for
use with our metrics solution. It also creates more opportunities for failures
that will be difficult for official support teams to diagnose.

While allowing this option is a true convenience for more advanced users, it
also pulls the project further from the "easy button" model of opinionated
deployment. As such, not implementing this feature should be considered as
a viable alternative.

## Affected Components

* oshinko-rest
* oshinko-console

## Testing

This feature should be tested through the functional client testing in the
oshinko-rest server.

Likewise, the oshinko-console extension should add these testing paths to
its functional tests (when available).

These tests should confirm the deployment of a custom image, and potentially
running an application against it (when this testing functionality is
available).

## Documentation

This feature will be added to the documentation for the oshinko-rest server,
and console extension.

