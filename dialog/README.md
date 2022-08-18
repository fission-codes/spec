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

## 1.1 Motivation

## 1.2 Goals & Use Cases

## 1.3 Properties

### 1.3.1 Subjective

### 1.3.1 Bitemporal

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

Each store has a single writer, which is represented by the signing authority of one key. While this does impose a single threaded writer, a single writer may write unboundedly large transactions as IPLD. This is done for a few reasons:



### 3.1.1 Concurrent Writes

## 3.2 Read Access

  
# 4 Collaboration

  * Merge Semantics
  * Unmerge semantcis
  
# 5 Query Engine

  * Differential Datalog

# 6 High-Level Language

  * Semantics 
  * Syntax

