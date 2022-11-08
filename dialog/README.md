# Dialog Specification v0.1.0 (Draft)

## Editors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Specs

* [Dataflow Runtime](./dialog/dataflow.md)
* [Query Engine](./dialog/query-engine.md)
* [Relational Algebra](./dialog/relational-algebra.md)

## Appendices

* [Research](./RESEARCH.md)

## Links

* [Fission Reactor: Dialog First Look](https://fission.codes/blog/fission-reactor-dialog-first-look/)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).


# Abstract

Dialog is a content-addressable database designed for use in far-edge deployments, such as on IoT or consumer devices. It targets this use case by enabling peers to synchronize heterogenous sets of end-to-end encrypted data in an eventually consistent manner.

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

# 2. Design

Dialog can be broken up into two core components: the query engine, and its storage layer.

## 2.1 Query Engine
Dialog has no specified query language. Instead, an intermediate representation based on the [relational algebra](../dialog/dialog/relational-algebra.md) is defined.

An OPTIONAL [Datalog variant](../dialog/dialog/query-engine.md) is described, along with an [algorithm](../dialog/dialog/relational-algebra.md#5-compilation-from-dialog) for translating it to the relational algebra IR.

Implementations MAY define their own user-facing query language, but they are RECOMMENDED to treat the relational algebra IR as a common compilation target for all such languages.

TODO: "relational algebra IR" is wordy. We should just give these layers names. Maybe PomoLogic -> PomoRA -> PomoFlow?

TODO: Talk about compiling SQL to PomoRA too. That should probably be another linked spec, but I should tackle one thing at a time.

This IR is then compiled to a form suitable for being evaluated by an implementation-defined runtime. An OPTIONAL [dataflow runtime](../dialog/dialog/dataflow.md) is described, but implementations MAY implement a simpler runtime, based on semi-naive evaluation.

TODO: Should we describe a simple specification for semi-naive evaluation yet, or just link to some resources? We'll definitely want to specify that too, at some point, but I'd prefer specifying one target to start, so we can launch sooner.

## 2.2 Storage

TODO: Introduce + link Brooke's upcoming work on persistence + encryption