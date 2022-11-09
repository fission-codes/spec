# PomoDB v0.1.0 (Draft)

## Editors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Specs

* [PomoFlow](./pomo_db/pomo_flow.md)
* [PomoLogic](./pomo_db/pomo_logic.md)
* [PomoRA](./pomo_db/pomo_ra.md)

## Appendices

* [Research](./RESEARCH.md)

## Links

* [Fission Reactor: Dialog First Look](https://fission.codes/blog/fission-reactor-dialog-first-look/)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).


# Abstract

PomoDB is a content-addressable database designed for use in far-edge deployments, such as on IoT or consumer devices. It targets this use case by enabling peers to synchronize heterogenous sets of end-to-end encrypted data in an eventually consistent manner.

# 1. Introduction

## 1.1 Motivation

While existing databases increasingly treat network partitions as unavoidable disturbances of a network, many mitigate these disturbances under a model that either assumes they'll be bounded in length, or by allowing them to impact the availability of some nodes in the network.

Such techniques are unsuitable for far-edge and local-first applications, where disorder is the norm, network partitions are ubiquitous and unbounded, and there's an uncompromising need for availability. These environments also often involve dynamic network topologies made up of heterogenous peers, for which common definitions of consistency may not apply.

PomoDB addresses these constraints through its query engine, whose semantics guarantee eventual consistency across peers with access to the same sources of data. These guarantees are preserved through changes to a peer's access to data, and mean that PomoDB is able to act as a sound foundation for globally distributed data with an indeterminate number of transient peers with varied access patterns.

## 1.2 Insights

TODO: discuss single system image / subjectivity

TODO: discuss CRDTs / BFT-CRDTs

TODO: discuss location independence

TODO: discuss CALM Theorem and its connection with Datalog

# 2. Design

## 2.1 Types

PomoDB is designed to be run within WebAssembly, and so its types and their encodings are informed by the format. See the [WebAssembly specification](https://webassembly.github.io/spec/core/appendix/index-types.html) for more information.

Note, however, that only some types are supported as PomoDB primitives. These are:

- [Numbers](https://webassembly.github.io/spec/core/syntax/types.html#syntax-numtype)
- [Opaque Reference Types](https://webassembly.github.io/spec/core/syntax/types.html#reference-types)

As WebAssembly does not define common types like booleans or strings, these are handled using opaque reference types, and more information is available in the [serialization](./serialization.md) specification.

TODO: Update the above reference once that data exists

## 2.2 Query Engine

PomoDB has no specified query language. Instead, an intermediate representation based on the relation algebra, named [PomoRA](./pomo_db/pomo_ra.md), is defined.

An OPTIONAL Datalog variant, named [PomoLogic](./pomo_db/pomo_logic.md), is described, along with an [algorithm](./pomo_db/pomo_ra.md#5-compilation-from-pomo-logic) for translating it to PomoRA.

Implementations MAY define their own user-facing query language, but they are RECOMMENDED to treat PomoRA as a common compilation target for all such languages.

TODO: Talk about compiling SQL to PomoRA too. That should probably be another linked spec, but I should tackle one thing at a time.

This IR is then compiled to a form suitable for being evaluated by an implementation-defined runtime. An OPTIONAL dataflow runtime, named [PomoFlow](./pomo_db/pomo_flow.md), is described, but implementations MAY implement a simpler runtime, such as one based on semi-naive evaluation.

TODO: Should we describe a simple specification for semi-naive evaluation yet, or just link to some resources? We'll definitely want to specify that too, at some point, but I'd prefer specifying one target to start, so we can launch sooner.

## 2.3 Storage

TODO: Introduce + link Brooke's upcoming work on persistence + encryption