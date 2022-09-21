# Dataflow Runtime

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Abstract

This document provides a recommendation for implementing Dialog's query engine in terms of a dataflow based runtime. This design is especially suited for incrementalizing programs to efficiently compute over deltas to the input EDB.

# 1. Introduction

Most Datalog runtimes implement a variant of semi-naive evaluation, where queries are run over the course of multiple iterations, with each iteration refining the result toward a fixed point. Such designs can efficiently compute views over a fixed EDB, but are insufficient for incrementaling evaluation over an EDB that changes over time.

This document describes an alternative runtime, built on dataflow, which represents programs as circuits whose vertices and edges correspond to computations over streams of data. These circuits can be incrementalized to instead operate over deltas, with the results being combined back into a materialized view.

Since the stream computations correspond to the relational algebra, compiling to these circuits from Datalog (and thus Dialog) is simple, and is also described in this document.

The design is based on ideas from Differential Dataflow, and is heavily inspired by the Database Stream Processor Framework (DBSP). Links to both can be found in the [research appendices](./RESEARCH.md).

# 2. Concepts

## 2.1 ZSet

A set of elements, each associated with a [weight](#24-weight) and [timestamp](#25-timestamp).

ZSets can be written as lists of triples:

```
[
    {element1, timestamp1, weight1},
    {element2, timestamp2, weight2},
    ...
]
```

Implementations should support efficient sequential access of a ZSet, first by element, and then by timestamp.

## 2.2 Indexed ZSet

A set of key-value pairs, each associated with a [weight](#24-weight) and [timestamp](#25-timestamp).

Indexed ZSets can be written as lists of triples:

```
[
    {{key1, value1}, timestamp1, weight1},
    {{key2, value2}, timestamp2, weight2},
    ...
]
```

Implementations should support efficient sequential access of an Indexed ZSet, first by key, and then by value, and lastly by timestamp.

## 2.3 Trace

A collection of deltas, combining multiple ZSets or Indexed ZSets.

Traces can be written as lists of 4-tuples:

```
[
    {key1, value1, timestamp1, weight1},
    {key2, value2, timestamp2, weight2},
    ...
]
```

Indexed ZSets should be merged into a trace using the key, value, timestamp, and weight associated with each element. ZSets should be merged into the trace by treating the element itself as a key, and associating no value with the delta.

Implementations MUST support efficient sequential access of traces, first by key, and then by value, and lastly by timestamp.

## 2.4 Weight

An integer associated with each element of a ZSet. Positive weights indicate the number of derivations of that element within the ZSet, and negative weights indicate the number of deletions of that element.

## 2.5 Timestamp

Every iteration of a [circuit](#26-circuit) is associated with a timestamp, and computed values are associated with the timestamp from which they were derived.

Timestamps MUST be represented using a partial order, such that subsequent iterations of the same [circuit](#26-circuit) have subsequent timestamps, and the timestamps of recursive subcircuits are given by refinements of the parent circuit's timestamp that allow iterations of that subcircuit to be distinguished and filtered according to that timestamp, called its epoch.

Implementations MAY represent timestamps for the root circuit as an integer, and timestamps for subcircuits as a pair constructed from their parent's timestamp, and an integer, and ordered using product order.

For example, a root circuit may progress through the following timestamps:

```
[0, 1, 2, 3, ...]
```

Whereas a subcircuit under that root may progress through these:

```
[
    (0, 0),
    (0, 1),
    (0, 2),
    (1, 0),
    (1, 1),
    (1, 2),
    ...
]
```

Under this example, using product order, `(0, 0) <= (0, 1) <= (1, 1)`, but neither `(0, 2) <= (1, 1)` or `(1, 1) <= (2, 0)`. In all of these cases, the epoch of each timestamp is the first component of the pair, and the iteration is given by its second.

## 2.6 Circuit

A circuit is an embedding of a Dialog program into a directed graph whose vertices, called [nodes](#28-node), represent computation against [streams](#27-stream), and whose edges describe those streams.

Circuits may contain subcircuits, representing recursive subcomputations that evaluate to a fixed point every iteration.

Each iteration, a circuit is evaluated by evaluating its nodes in topological order, until each has reached a fixed point.

## 2.7 Stream

A stream is an infinite sequence of values, each associated with subsequent timestamps. It is RECOMMENDED that streams not be reified directly, and they MAY instead be modeled as a cell containing the singular value in the stream at the current timestamp.

The edges between nodes in a circuit describe streams of values flowing from the output of one node to the input of another, and these edges SHOULD be defined in terms of the IDs of the nodes they connect.

## 2.8 Node

A node is a vertex of a circuit, and describes a computation over the circuit's streams.

Every node is associated with both a local ID and a global ID. Local IDs MUST be unique among all nodes belonging to the same circuit, and global IDs MUST be unique across all nodes.

It is RECOMMENDED that local IDs be represented by a node's index into its parents nodes, and that global IDs be represented as the path formed by the local IDs of a node's ancestors.

| Type                           | Inputs | Outputs |
| ------------------------------ | ------ | ------- |
| [Operator](#281-operator-node) | N-ary  | 1       |
| [Child](#282-child-node)       | 0      | N-ary   |
| [Feedback](#283-feedback-node) | 1      | 1       |
| [Import](#284-import-node)     | 1      | 1       |
| [Sink](#285-sink-node)         | N-ary  | 0       |
| [Source](#286-source-node)     | 0      | 1       |

## 2.8.1 Operator Node

Performs an [operation](#29-operator) against its input streams, outputting the result over a stream.

Operator nodes achieve a fixed point when their associated operator has done so.

## 2.8.2 Child Node

Introduces a subcircuit that can be used to perform recursive computations. Such circuits evaluate to a fixed point each iteration, using the same rules as the root circuit, then emit their result to downstream nodes.

## 2.8.3 Feedback Node

Introduces a temporal edge between nodes in subsequent iterations of a circuit. Such nodes support persisting data between iterations of a circuit, and are also used to implement recursive circuits by propagating partial results forward in time until a fixed point is reached.

These nodes introduce apparently cycles into a circuit, however implementations SHOULD NOT treat them as such, and SHOULD consider their output stream to be lazily evaluated as part of the subsequent iteration.

Intuitively, these nodes can be thought of as delaying a stream by one iteration.

TODO revisit this section after I explain operators better

## 2.8.4 Import Node

Imports a parent stream into a subcircuit.

TODO explain delta0 + mention timestamp refinement

## 2.8.5 Sink Node

Performs an operation against its input streams, without outputting anything.

## 2.8.6 Source Node

Emits data over a stream, without accepting any inputs from the circuit.

## 2.9 Operator

An operator specifies an operation against a stream, and is represented by a node. Operators MAY be stateful, and linear operators are those which can be computed using only the deltas at the current timestamp.

| Name                                                    | Linearity              | Input Types                | Output Type  |
| ------------------------------------------------------- | ---------------------- | -------------------------- | ------------ |
| [Aggregate](#291-aggregate-operator)                    | Varies                 | Indexed ZSet               | ZSet         |
| [Consolidate](#292-consolidate-operator)                | Linear                 | Trace                      | ZSet         |
| [Distinct](#293-distinct-operator)                      | Non-Linear             | ZSet                       | ZSet         |
| [Filter](#294-filter-operator)                          | Linear                 | ZSet                       | ZSet         |
| [IndexWith](#295-indexwith-operator)                    | Linear                 | ZSet                       | Indexed ZSet |
| [Inspect](#296-inspect-operator)                        | Linear                 | ZSet                       | ZSet         |
| [Map](#297-map-operator)                                | Linear                 | ZSet                       | ZSet         |
| [Negate](#298-negate-operator)                          | Linear                 | ZSet                       | ZSet         |
| [Z1Trace](#299-z1trace-operator)                        | Linear                 | Trace                      | Trace        |
| [Z1](#2910-z1-operator)                                 | Linear                 | ZSet                       | ZSet         |
| [DistinctTrace](#2911-distincttrace-operator)           | Non-Linear             | ZSet, Trace                | ZSet         |
| [JoinStream](#2912-joinstream-operator)                 | Linear                 | Indexed ZSet, Indexed ZSet | ZSet         |
| [JoinTrace](#2913-jointrace-operator)                   | Bilinear               | Indexed ZSet, Trace        | ZSet         |
| [Minus](#2914-minus-operator)                           | Linear                 | ZSet, ZSet                 | ZSet         |
| [Plus](#2915-minus-operator)                            | Linear                 | ZSet, ZSet                 | ZSet         |
| [TraceAppend](#2916-traceappend-operator)               | Non-Linear (TODO ish?) | ZSet, Trace                | Trace        |
| [UntimedTraceAppend](#2917-untimedtraceappend-operator) | Non-Linear (TODO ish?) | ZSet, Trace                | Trace        |

## 2.9.1 Aggregate Operator

Applies an aggregate to all elements of the input stream.

## 2.9.2 Consolidate Operator

Merges all ZSets in the input trace.

## 2.9.3 Distinct Operator

Returns one of each element whose summed weight is positive.

## 2.9.4 Filter Operator

Filters a ZSet by a predicate.

## 2.9.5 IndexWith Operator

Groups elements of a ZSet according to some key function.

## 2.9.6 Inspect Operator

Applies a callback to a ZSet, returning the original ZSet.

## 2.9.7 Map Operator

Transforms elements of a ZSet according to some function.

## 2.9.8 Negate Operator

Negates the weights of each element in a ZSet.

## 2.9.9 Z1Trace Operator

Returns the previous input trace.

## 2.9.10 Z1 Operator

Returns the previous ZSet.

## 2.9.11 DistinctTrace Operator

Incremental version of Distinct (TODO separate this out?).

## 2.9.12 JoinStream Operator

Joins two ZSets together according to their keys (TODO might not want this).

## 2.9.13 JoinTrace Operator

Incremental version of JoinStream (TODO separate this out?).

## 2.9.14 Minus Operator

Subtracts all weights for matching keys in two ZSets.

## 2.9.15 Plus Operator

Adds all weights for matching keys in two ZSets.

## 2.9.16 TraceAppend Operator

Inserts the input ZSet into the input Trace, with the current timestamp.

## 2.9.17 UntimedTraceAppend Operator

Inserts the input ZSet into the input Trace (TODO maybe not needed anymore).
