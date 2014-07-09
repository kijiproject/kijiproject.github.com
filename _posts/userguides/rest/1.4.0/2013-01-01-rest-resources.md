---
layout: post
title: KijiREST Resources
categories: [userguides, rest, 1.4.0]
tags: [rest-ug]
order: 3
version: 1.4.0
description: Resources
---

Every REST request is parametrized by a resource path that uniquely identifies a server-side resource.
The REST identifiers are intended to allow clients to intuitively consume server-side resources.

#### /v1

The KijiREST API version is placed at the root of the resource path as the API entry point.
Prefixing the resource path with the version in this manner results in graceful upgrades.

    /v1/[...]

#### /v1/instances

A Kiji cluster contains Kiji instances. To avoid name clashes with other direct child
resources of the Kiji cluster resource (that is, with the version resource), all instances
are grouped into a sub-collection named "instances". This resource serves as a namespace for
all instances on this cluster.

    /v1/instances/

#### /v1/instances/&lt;instance&gt;

To access a particular instance, request it as a child resource of the "instances" collection:

    /v1/instances/<instance>

#### /v1/instances/&lt;instance&gt;/tables

Every instance contains a collection named "tables". It serves as a namespace for all the tables in this instance.

    /v1/instances/<instance>/tables/

#### /v1/instances/&lt;instance&gt;/tables/&lt;table&gt;

To access a particular table, request it as a child resource of the "tables" collection:

    /v1/instances/<instance>/tables/<table>

#### /v1/instances/&lt;instance&gt;/tables/&lt;table&gt;/rows

Every table contains a collection named "rows".

    /v1/instances/<instance>/tables/<table>/rows/

#### /v1/version

Every KijiREST service is associated with a single Kiji cluster and the cluster may be
considered the "root" resource (after the REST API version). The Kiji cluster's version
(distinct from the KijiREST API version) is the endpoint within the cluster:

    /v1/version

### The File System Directory Tree Analogy

The Kiji resource chain induces a simple directory tree analogy where directories represent
collections of resources.

For example, consider a cluster where there are two instances named "dev_instance" and
"prod_instance". The prod_instance contains two tables, which are named "customers" and
"songs":

![REST Resources analogous to Directory Tree][kiji-rest-hierarchy]

[kiji-rest-hierarchy]: ../../../../assets/images/kiji-rest-hierarchy.png


