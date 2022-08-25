# Dialog Query Engine

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Abstract

Dialog is query language agnostic, and implementations MAY define arbitrary query frontends. This specification describes the semantics for Dialog's underlying query engine, to which these frontends MUST conform.

# 1 Introduction

The semantics of Dialog's underlying query engine match those of Datalog, extended to support queries over time against a dynamic and content-addressable database.

By restricting the expressivity of the engine in this way, it's able to take advantage of the CALM Theorem to ensure the confluence of any programs written using the system.

Datalog is a relational query language that operates on a database of tuples, named atoms, using programmer defined rules, in order to compute a view over that database.

The following section serves as a brief introduction to Datalog, but a comprehensive description of the language and its various extensions is beyond the scope of this specification, and more complete treatments can be found in the [research appendices](./RESEARCH.md).

## 1.1 Notation

This section briefly describes a high-level notation for Datalog, as it relates to Dialog, for the purposes of discussing the semantics of the system. An example DSL is [also available](./datalog-dsl.md).

### Terms

A Datalog term is either a constant, or a variable. Variables begin with an uppercase character, and constants are written as literals.

The allowed types are defined in [section 1.2](#types).

### Atoms

Atoms are tagged tuples of the form:

```datalog
foo(a1, a2, ..., an)
```

Where `foo` is the name of the atom, and `a1, a2, ..., an` are its arguments. The arguments to an atom can be any [term](#terms), and an atom whose arguments are all constants is called a ground atom.

For example, `point(3, 7)` is a ground atom named `point` with two arguments, `3` and `7`. Similarly, `point(X, Y)` is a non-ground atom with two variable arguments, `X` and `Y`.

An instance of a ground atom is often referred to as a tuple for brevity, and the collection of all tuples with a given name is called a relation.

### Database

Using standard Datalog terminology, Dialog operates over two databases: the extensional database, and the intensional database.

The extensional database (EDB) refers to the source database containing the ground tuples which act as inputs to a program. Similarly, the intensional database (IDB) refers to the ground tuples which are derived using the rules in a program, `P`.

Connecting both concepts is the Herbrand base of `P`, containing all ground tuples in either the EDB or IDB:

```
B(P) = EDB ∪ IDB(P)
```

### Rules

Rules are written:

```datalog
A :− B1, B2, ..., Bn.
```

Where `A` is an [atom](#atoms) that forms the head of the rule, and `B1, B2, ..., Bn` are [predicates](#predicates) that make up its body. Such a rule can be understood to mean that an instance of `A` should be added to the database if all of the predicates given by `B` through `Z` are true.

The head of a rule may contain variables, in which case a successful application of the rule's body will instantiate them with constants. For example, the following rule projects the first argument of all instances of the `point` relation, to produce matching `xCoordinate` tuples:

```datalog
xCoordinate(X) :− point(X, Y).
```

Such a rule can also be written using wildcards in the place of unused variables:

```datalog
xCoordinate(X) :− point(X, _).
```

Rules may also have multiple heads, each with their own body. The following rule is defined using two heads which together compute `zeroPoint` tuples with either their first or second argument being `0`:

```datalog
zeroPoint(0, Y) :− point(0, Y).
zeroPoint(X, 0) :− point(X, 0).
```

Importantly, valid rules are those which are range restricted, where all variables in the head appear within the body as part of a [selection predicate](#predicates).

All rules discussed so far are examples of deductive rules. Similarly inductive rules are defined by suffixing a rule's head with `@next`:

```datalog
zeroPoint(X, Y)@next :− zeroPoint(X, Y).
```

This causes the rule to delay derived tuples until the next timestep. For more information, see [section 1.3.2](#131-time).

### Predicates

A [rule's](#rules) body may contain a number of different predicates that determine the conditions under which that rule applies:


```
Predicate ::=
    Selection
  | Negation
  | Constraint
  | Var := Aggregation

Selection ::=
    Atom
  | Var := Atom

Negation ::=
    !Atom

Aggregation ::= count :
    Atom
  | sum Var : Atom
  | min Var : Atom
  | max Var : Atom
  | UserDefinedAggFun Var : Atom

Constraint ::=
    Term == Term
  | Term != Term
  | Term < Term
  | Term <= Term
  | Term > Term
  | Term >= Term
```

For example, the following program filters social media profiles to select those with at least 1000 followers:

```datalog
popularProfile(X) :−
    profile(X),
    c := count : follower(X, _),
    c >= 1000.
```

Here, `c` is bound to the count of `follower` tuples whose first argument matches `X`. The result is then constrained to only those variable bindings whose associated counts are greater than 1000.

Detailed descriptions of these predicates is provided in [section 1.3](#13-semantics).

## 1.2 Types

Dialog is designed to be run within WebAssembly, and so its types and their encodings are informed by the format. See the [WebAssembly specification](https://webassembly.github.io/spec/core/appendix/index-types.html) for more information.

Note, however, that only some types are supported as Dialog primitives. These are:

- [Numbers](https://webassembly.github.io/spec/core/syntax/types.html#syntax-numtype)
- [Opaque Reference Types](https://webassembly.github.io/spec/core/syntax/types.html#reference-types)

As WebAssembly does not define common types like booleans or strings, these are handled using opaque reference types, and more information is available in the [serialization](./serialization.md) specification.

## 1.3 Semantics

Dialog follows a fixed point semantics for Datalog, with stratified negation and aggregation, and extensions for reifying time and content IDs.

## 1.3.1 Time

Unlike many variants of Datalog, Dialog operates over a changing database, and the query engine internally models the progression of time as the database changes. Importantly, this notion of time forms a logical clock which holds no meaning to any other instances of Dialog.

The database state is then modeled as a function of time and the program under evaluation, but this computation is intentionally left unspecified, and implementations are free to make their own choices according to their performance constraints and desired features, subject to the following semantics.

First, let `P` be some Dialog program, let `next(t)` denote the smallest timestamp `t'`, such that `t < t'`, and let `IDB(t, p)` denote the IDB at time `t`.

Then, for all deductive rules `r` in `P`, if `r` derives a tuple `X` at time `t`, then `X` is in `IDB(t, p)`. Similarly, for all inductive rules `r` in `P`, if `r` derives a tuple `X` at time `t`, then `X` is in `IDB(next(t), P)`.

In order to guarantee a finite execution time for programs containing inductive rules, a few constraints are imposed over their use. In particular, to avoid programs which change infinitely over time for a static EDB, one of two properties must hold for each inductive rule:

1) The head atom appears in the body with the same bindings
2) The body contains at least one positive instantaneous predicate
   1) TODO: Explain what this means!

Treating time in this way is what allows Dialog to support both transient tuples—like those that might be used to model events in a user interface—and persistent tuples that capture long-lived data.

For example, given a relation `clicks` whose tuples denote a user clicking on a UI element during the current timestamp, a user interface containing a checkbox might be modeled like so:

```datalog
checkbox(Id, S)@next     :- checkbox(Id, S), !clicks(Id).
checkbox(Id, true)@next  :- checkbox(Id, false), clicks(Id).
checkbox(Id, false)@next :- checkbox(Id, true), clicks(Id).
```

This rule has three heads which together relate the current state of the checkbox to its state in the next timestamp. The first handles the case where no click has occurred, by simply copying the current state forward in time, whereas the next two handle a click event by flipping the checkbox's state at the next timestamp.

Note that while this rule depends negatively on itself, it's able to do so safely because the timestamp of the head is larger than the timestamps of the atoms in the body, modularly stratifying the program with respect to time.

## 1.3.2 Content Addressing

As Dialog is intended for use in distributed and decentralized deployments, it is important ensure the use of collision resistant identifiers when referring to tuples. For this purpose, a content addressing scheme is leveraged, wherein facts are associated with a content ID (CID) computed from their structure. The details behind this computation are available in [serialization](./serialization.md).

This CID is accessed through the selection predicate, which can optionally bind a tuple's CID to a variable:

```datalog
C := point(X, Y)
```

In this way, these CIDs can be exposed to rules for use in joins between relations, or in the derivation of new tuples.

For example, the following rule operates over an EDB containing tuples which describe the product offerings of an e-commerce site, and computes the number of products belonging to each category:

```datalog
categoryCount(Category, Count) :-
    Category := category(...),
    Count := count : product(Category, ...).
```

In this example, the CID of each tuple in the `category` relation acts as its primary key, and is suitable for use as a foreign key when joining against this relation for aggregation purposes.

The choice of CIDs here rather than more common choices, like autoincrementing IDs or UUIDs, reflects Dialog's goals in targeting distributed and decentralized environments, where coordination around the allocation of IDs can't be guaranteed, and where resilience against malicious and byzantine actors is required.

Since content addressing schemes are backed by cryptograpically secure hash functions, their use here prevents forgery of IDs by attackers, and guarantees that CID-based dependencies between tuples will be acyclic.

These properties are further leveraged in the design and use of byzantine-fault tolerant CRDTs, as described in [CRDTs](./CRDTs.md).

## 1.3.3 Stratification

Dialog supports recursion, negation, and aggregation, and in order to do so safely, it makes use of stratification. Stratification partitions a program's rules into a sequence of strata, based on their dependencies, such that all rules within a stratum depend on each other. These strata are then ordered such that no stratum appears before a stratum containing rules it depends on.

This gives an evaluation order for the program which guarantees convergence toward a unique IDB for any evaluation of a given EDB at some time.

A program may have multiple valid stratifications, if one exists, but the choice of stratification does not matter for the system's semantics, and a suitable choice can be computed through the following algorithm:

1) For some program `P`, construct its precedence graph, `G`:
   1) For each atom `A` appearing in `P` , add `A` as a vertex to `G`
   2) For each deductive rule `r` in `P`, with head atom `H` and a selection predicate against atom `B`, add `(H, B)` as a directed edge labelled as having positive polarity to `G`
   3) For each deductive rule `r` in `P`, with head atom `H` and a negation predicate against atom `B`, add `(H, B)` as a directed edge labelled as having negative polarity to `G`
   4) For each deductive rule `r` in `P`, with head atom `H` and an aggregation predicate against atom `B`, add `(H, B)` as a directed edge labelled as having negative polarity to `G`
2) Compute the condensation of `G`, `C(G)`:
   1) Calculate the strongly connected components (SCCs) of `G`, i.e. the maximal subgraphs of `G` such that there exists a path between every pair of vertices, in both directions
   2) Contract each strongly connected component in `G` to a single vertex, and add it to `C(G)`
   3) For any two distinct strongly connected components in `G`, `c1` and `c2`, if there exists an edge from a vertex in `c1` to a vertex in `c2`, then add a directed edge between the corresponding vertices for `c1` and `c2` in `C(G)`
3) Perform a topological sort on `C(G)`: the resulting ordering gives the stratification of `P`, with each vertex in `C(G)`, `c`, corresponding to a stratum containing the rules in `P` whose head belongs to the strongly connected component of `G` for which `c` is associated
4) Append a new stratum to the end, containing all inductive rules. This last stratum corresponds to the modular stratification over time discussed in [section 1.3.1](#131-time)

Such a stratification is only valid if for every strongly connected component in `G`, no edge within that component is labelled with negative polarity. Intuitively, this prevents the use of negation or aggregation through recursive application of rules. Such programs are considered unstratisfiable, and cannot be represented using Dialog.

## 1.3.4 Evaluation