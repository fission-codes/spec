# Dialog Specification v0.1.0

## Editors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Abstract

Dialog is a local-first database with novel ownership semantics, collaboration, and fault tollerance.

# 1 Introduction

## 1.1 Motivation & Philosophy

## 1.2 Goals & Use Cases

### 1.2.1 User-Controlled Apps

Should be easy for FE devs [...]

### 1.2.2 Soft Realtime Collaboration

Docs

### 1.2.3 Blind Busses

# 1.3 Properties

### 1.3.1 User Agency

### 1.3.1 Subjectivity

Dialog eschews the [Single System Image](https://arxiv.org/pdf/1510.08473.pdf) in favor of [subjective views](https://people.csail.mit.edu/malte/pub/papers/2019-hotos-multiversedb.pdf), interpretations, and reinterpretations(!) of data. This embraces the fact that information is not evenly geograhicalluy distributed, that different users are interested in different data, and that social trust varies across users and time.

This also leads to bitemporality.

### 1.3.2 Tunable Consistency

### 1.3.2 Byzantine Fault Tollerance

### 1.3.1 Location Independence

### 1.3.3 Portablility

## 1.4 High Level Structure

FIXME diagram here (stores, collections, connections, etc)

# 2 Data Layer

  * Fact structure
  * Store structure
  * Serialization 
  * WNFS embedding
  
# 3 Access Control

Access control within each Datalog store is intended to be extremely simple, and to get out of the way of the core engine as much as possible.

[WebNative File System](https://github.com/wnfs-wg/spec), 

## 3.1 Write Access

Each store has a single writer, which is represented by the signing authority of one key. While this does impose a single threaded writer, a single writer may write unboundedly large transactions as IPLD. This leads to a pull-based sychronization mechanism. This is done for a few reasons:


### 3.1.1 Concurrent Writes

## 3.2 Read Access

<!-- Abstract  -->

Access patterns are varied and complex. Placing some restriction on 

Read access control MUST be accomplished by encrypting the fact and indices at rest.

### 3.2.1 Table Heirarchy Pattern

* Backward Secrecy (Ratchet)

Compound ratchet components: Store, entity, attribute, [FIXME hmmm value needs a unique key, and global counters are really bad for lookups...]
Compound ratchet heirarchy: Store -> entity -> attribute -> value

Used _both_ for lookup and encryption

Minor revocations by tombstoning (forward posting), ratcheting upper portion + zero cascading. Evicting the entire store via sealing store with tombstone, and creating a new store.

```
┌────────────Store───────────┐
│                            │
│  ┌─────────Entity───────┐  │
│  │                      │  │
│  │   ┌───Attribute───┐  │  │
│  │   │               │  │  │
│  │   │               │  │  │
│  │   │     Value     │  │  │
│  │   │               │  │  │
│  │   │               │  │  │
│  │   └───────────────┘  │  │
│  │                      │  │
│  └──────────────────────┘  │
│                            │
└────────────────────────────┘
```

```
┌─────────Store──────────┐             ┌─────────Store─────────┐
│                        │             │                       │
│  ┌─────────────────────┼───Entity────┼────────────────────┐  │
│  │                     │             │                    │  │
│  │   ┌─────────────────┼──Attribute──┼─────────────────┐  │  │
│  │   │                 │             │                 │  │  │
│  │   │  ┌───────────┐  │             │  ┌───────────┐  │  │  │
│  │   │  │           │  │             │  │           │  │  │  │
│  │   │  │  Value A  │  │             │  │  Value B  │  │  │  │
│  │   │  │           │  │             │  │           │  │  │  │
│  │   │  └───────────┘  │             │  └───────────┘  │  │  │
│  │   │                 │             │                 │  │  │
│  │   └─────────────────┼─────────────┼─────────────────┘  │  │
│  │                     │             │                    │  │
│  └─────────────────────┼─────────────┼────────────────────┘  │
│                        │             │                       │
└────────────────────────┘             └───────────────────────┘
```

### 3.2.2 Graph Dependency Pattern

Graph-structured data often requires history to make sense of the later values. Two illustrative examples are LWW registers and [BFT-RGA](https://martin.kleppmann.com/papers/bft-crdt-papoc22.pdf):

* LWW registers only watch the latest value, but determining this value depends on having access to _at minimum_ the greatest common meet
* BFT-RGA encodes the structure of a docuemnt directly as linked dependencies between values that are nonsense on their own. At least a single full path (starting from genesis) through the graph is required

This implies that dependency links MUST include the key to the previous fact _without_ exposing the internal ratchet state. "Flattened" skip ratchet

Later updates MUST also be accessible to 

Revocation -- especialy across stores -- is really difficult in this model, as other stores may continue to write in to the same history without realizing it. Incluing revocation lists as part of tombstoning may be one one way to communiate this info [FIXME!]

#### 3.2.2.1 Hidden Branches

Not all readers should be able to read all branches. This is accomplished by chaning one of the ratchet levels or a salt [FIXME expand!] and using blank nodes for attachment points.

```
 Public  Blank    Private
┌──────┐ ┌───┐ ┌───────────┐
 ○───►○───►◘───►●───►●───►●

 
 ○───►○───►◘───►●───►●───►●
           │
           └───►○───►○───►○

 
           ┌───►●───►●───►●
           │
 ○───►○───►◘
           │
           └───►○───►○───►○

 
 ##############
 ## OPPOSITE ##
 ##############

 
 ○───►○───►○───►○───►○───►○
 

           ┌───►●───►●───►●
           │
 ○───►○───►○───►○───►○───►○ # NOTE no blank/pivot/anchor node

 
           ┌───►●───►●───►●
           │
 ○───►○───►○
           │
           └───►○───►○───►○
```

### 3.2.3 Interaction
  
# 4 Collaboration

  * Merge Semantics
  * Unmerge semantcis
  
# 5 Query Engine

  * Differential Datalog

# 6 High-Level Language

  * Semantics 
  * Syntax

