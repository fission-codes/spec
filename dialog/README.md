# Dialog Specification v0.1.0

## Editors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Specs

* [Query Engine](./dialog/query-engine.md)

## Appendices

* [Research](./RESEARCH.md)

## Links

* [Fission Reactor: Dialog First Look](https://fission.codes/blog/fission-reactor-dialog-first-look/)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).


# Abstract

Dialog is a content-addressable database designed for use in far-edge deployments, such as on IoT or consumer devices. It targets this use case by enabling peers to synchronize hterogenous sets of end-to-end encrypted data in an eventually consistent manner.

# 1. Introduction

## 1.1 Motivation

While existing databases increasingly treat network partitions as unavoidable disturbances of a network, many mitigate these disturbances under a model that either assumes they'll be bounded in length, or by allowing them to impact the availability of some nodes in the network.

Such techniques are unsuitable for far-edge and local-first applications, where disorder is the norm, network partitions are ubiquitous and unbounded, and there's an uncompromising need for availability. These environments also often involve dynamic network topologies made up of heterogenous peers, for which common definitions of consistency may not apply.

Dialog addresses these constraints through its query engine, whose semantics guarantee eventual consistency across peers with access to the same sources of data. These guarantees are preserved through changes to a peer's access to data, and mean that Dialog is able to act as a sound foundation for globally distributed data with an indeterminate number of transient peers with varied access patterns.

## 1.2 Insights

TODO: discuss single system image / subjectivity

TODO: discuss CRDTs / BFT-CRDTs

TODO: discuss location independence

TODO: discuss CALM Theorem and its connection with Datalog