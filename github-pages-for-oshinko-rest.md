# Example Spec - title

## Introduction

The documentation for oshinko-rest is currently spread across a few files
that are distributed throughout the project. This specification proposes
bringing all the docs into a GitHub Pages oriented layout that will enable
a more user friendly documentation experience.

## Problem statement

Currently the documentation for the oshinko-rest server is held in several
files with little linking between them. This creates a small amount of
confusion when scanning the files looking for specific information. An
interim solution of placing the files into a `docs` directory has helped,
but further improvements could be made to make their navigation easier.

## Proposed solution

To address the issue of creating a more complete documentation experience,
the `docs` directory should be converted to use the
[Jekyll](http://jekyllrb.com/) framework in conjunction with the
[GitHub Pages](https://pages.github.com/) deployment option.

Creating a true web-based approach to the documentation will allow deeper
interactions between the pages available, and provide a solid starting point
for referencing the tip of the documentation. Additionally, as Jekyll can
use Markdown as a primary format, many of the current documents will convert
quickly to the new framework.

A [Patternfly](http://www.patternfly.org/) based web scaffolding will be
used to create a similar look and feel to the OpenShift environment and
the user interface experiences that are associated with Oshinko.

The primary readme file will be updated to reflect the new documentation
location.

### Alternatives

One alternative would be to simply consolidate the current documentation into
the docs directory and ensure that they are linked properly through the use
of GitHub links. This solution would be faster and require less setup work
with the web framework, but would use the standard GitHub interface to
display the Markdown files.

## Affected Components

* oshinko-rest repository

## Testing

In general this should not need testing beyond the basic deploy mechanics
provided by GitHub Pages.

## Documentation

A meta-document describing the process for adding new documentation will be
added to the `docs` directory, this may also be linked from the generated
documentation.
