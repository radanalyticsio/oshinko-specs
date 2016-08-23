# Functional end-to-end testing

## Introduction

This specification details a process and workflow for implementing
functional end-to-end testing for the oshinko-rest-server application.
These are tests which run in a live environment to demonstrate the full
functionality of the server.

## Problem statement

Currently, the oshinko-rest-server application only contains unit testing for
some of its functions. Although these tests are valuable, they should be
supplemented by tests which can fully excercise all the features of the
server. By testing the server in a live environment, we can learn about gaps
between the application and a fully deployed OpenShift environment. We can
also use these tests as the foundation of a release oriented cycle that
requires valid results before publication.

## Proposed solution

The oshinko-rest repository carries a set of client tests that exist in
`tests/client`. These tests start a live server and use the Go client to
interact with the server. The tests are meant to be used in an offline
manner to ensure that all client routines work properly. Using the client
tests as a basis, a set of new tests shall be created to be used in an
environment running an OpenShift cluster.

The initial thrust of the functional tests should be focused on creating a
set of live tests which can use the oshinko-rest Go client to exercise all the
operations provided, and ensure that the results are consistent with expected
behavior. These will be positive tests that will attempt to manipulate the
oshinko-rest-server through a series a scripted examples to create, scale,
update, and destroy a Spark cluster.

Implementing the testing framework on top of the current Go tests will allow
great flexibility when deploying and exercising the tests. This flexibility
lends itself well to growing the testing infrastructure by allowing simple,
developer initiated testing to begin with, and then growing easily to fit
within a larger testing story involving a continuous integration server.

To aid in deploying the tests, a Kubernetes job template shall be created.
Use of this template will facilitate a simple deploy, run, receive results
loop that should be repeatable for any developer.

Once the initial series of tests have been implemented, the issue of
integrating into a larger infrastructure should be examined for availibility
and tooling. Ideally, this infrastructure will involve deploying oshinko-rest
into a fresh OpenShift Enterprise environment, running the tests, and
recording the resultant output. The longer term goal for this integration is
to enable both unit and functional testing on all pull requests that are
proposed to the github repository.

The details surrounding a larger infractructure integration should be
addressed in a later specification as this will require either creating our
own test stand or determining the proper structure to use another test stand.

### Alternatives

One alternative is to create a series of tests which can run using only the
REST interface. This would require more work to create the REST client, which
would ultimately be a replication of the client code we currently have. An
upside to this approach is that we could use non-Go based solutions to create
the REST client, this may provide more options for integrations in the future.

Another alternative is to not use the Go testing based tests that are
currently in use, and instead create a harness that will start the server and
then run through the scripted tests. This option may provide a wider range
of options for deploying and testing the server at the cost of not re-using
our currently implemented test structure.

## Affected Components

oshinko-rest

## Testing

Testing the test? A novel idea...

## Documentation

Documentation will be created for any new testing styles. This will describe
how to deploy and run the tests as well as how to interpret results and add
new tests.
