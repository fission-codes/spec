# PomoDB v0.1.0 (Draft)

## Editors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Specs

* [PomoFlow](pomo_db/pomo_flow.md)
* [PomoLogic](pomo_db/pomo_logic.md)
* [PomoRA](pomo_db/pomo_ra.md)

## Appendices

* [Research](RESEARCH.md)

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

As WebAssembly does not define common types like booleans or strings, these are handled using opaque reference types, and more information is available in the [serialization](pomo_db/serialization.md) specification.

TODO: Update the above reference once that data exists

## 2.2 Relation

A relation is a set of tuples, where each component of the tuple is called an attribute, and can be referenced by an attribute name.

A n-ary relation contains n-tuples, which can each be written:

```
(a1: v1, a2: v2, ..., an: vn)
```

Where `a1, a2, ..., an` give the names of each attribute, and `v1, v2, ..., vn` their values.

Each tuple within a relation also has a content identifier (CID), as described in the specification for [PomoLogic](pomo_db/pomo_logic.md#132-content-addressing). This CID can be accessed through a special control attribute that this document will denote as `$CID`: however implementations are RECOMMENDED to use their type system to differentiate between such attributes.

## 2.3 Content Addressing

As PomoDB is intended for use in distributed and decentralized deployments, it is important ensure the use of collision resistant identifiers when referring to tuples. For this purpose, a content addressing scheme is leveraged, and tuples are associated with a content ID (CID) computed from their structure. The details behind this computation are available in [serialization](pomo_db/serialization.md).

The choice of CIDs here, rather than more common choices, like auto incrementing IDs or UUIDs, reflects PomoDB's goals in targeting distributed and decentralized environments, where coordination around the allocation of IDs can't be guaranteed, and where resilience against malicious and byzantine actors is required.

Since content addressing schemes are backed by cryptographically secure hash functions, their use here prevents forgery of IDs by attackers, and guarantees that CID-based dependencies between tuples will be acyclic.

These properties are further leveraged in the design and use of byzantine-fault tolerant CRDTs, as described in [CRDTs](pomo_db/CRDTs.md).

TODO: Update CRDT link once that info is described somewhere

## 2.4 Provenance Tracking

TODO: https://discord.com/channels/478735028319158273/1033502043656171561/1035339517021921280

## 2.5 Query Engine

PomoDB has no specified query language. Instead, an intermediate representation based on the relation algebra, named [PomoRA](pomo_db/pomo_ra.md), is defined.

An OPTIONAL Datalog variant, named [PomoLogic](pomo_db/pomo_logic.md), is described, along with an [algorithm](pomo_db/pomo_ra.md#4-compilation-from-pomo-logic) for translating it to PomoRA.

Implementations MAY define their own user-facing query language, but they are RECOMMENDED to treat PomoRA as a common compilation target for all such languages.

TODO: Talk about compiling SQL to PomoRA too. That should probably be another linked spec, but I should tackle one thing at a time.

This IR is then compiled to a form suitable for being evaluated by an implementation-defined runtime. An OPTIONAL dataflow runtime, named [PomoFlow](pomo_db/pomo_flow.md), is described, but implementations MAY implement a simpler runtime, such as one based on semi-naive evaluation.

TODO: Should we describe a simple specification for semi-naive evaluation yet, or just link to some resources? We'll definitely want to specify that too, at some point, but I'd prefer specifying one target to start, so we can launch sooner.

## 2.6 Storage

TODO: Introduce + link Brooke's upcoming work on persistence + encryption