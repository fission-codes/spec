# PomoLogic

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Abstract

PomoDB is query language agnostic, and implementations MAY define arbitrary query frontends. This specification describes the semantics for PomoLogic, a variant of Datalog, designed for recursive query processing against PomoDB.

# 1. Introduction

TODO: Now that the query language is described in a few layers, some of these details should be pulled out of PomoLogic so they can be reused for non-recursive queries: PomoRA is probably the place for that.

The semantics of PomoLogic match those of Datalog, extended to support queries over time against a dynamic and content-addressable database.

By restricting the expressivity of the engine in this way, it's able to take advantage of the CALM Theorem to ensure the confluence of any programs written using the system.

Datalog is a relational query language that operates on a database of tuples, named atoms, using programmer defined rules, in order to compute a view over that database.

The following section serves as a brief introduction to Datalog, but a comprehensive description of the language and its various extensions is beyond the scope of this specification, and more complete treatments can be found in the [research appendices](./RESEARCH.md).

## 1.1 Notation

This section briefly describes a high-level notation for Datalog, as it relates to PomoLogic, for the purposes of discussing the semantics of the system. An example DSL is [also available](./datalog-dsl.md).

TODO: Either make this file or put that info elsewhere.

### Terms

A Datalog term is either a constant, or a variable. Variables begin with an uppercase character, and constants are written as literals.

The allowed types are defined in [section 1.2](#types).

### Atoms

Atoms are tagged tuples of the form:

```datalog
foo(a1: v1, a2: v2, ..., an: vn)
```

Where `foo` is the name of the atom, `a1, a2, ..., an` are its attribute names, and `v1, v2, ..., vn` are the values of those attributes. The values in an atom's attributes can be any [term](#terms), and an atom whose attributes all have constant values is called a ground atom.

For example, `point(x: 3, y: 7)` is a ground atom named `point` with two attributes, `x` with value `3` and `y` with value `7`. Similarly, `point(x: X, y: Y)` is a non-ground atom with two variable attributes, `x` and `y`.

An instance of a ground atom is often referred to as a tuple for brevity, and the collection of all tuples with a given name is called a relation.

### Database

Using standard Datalog terminology, PomoLogic operates over two databases: the extensional database (EDB), and the intensional database (IDB).

The extensional database refers to the source database containing the ground tuples which act as inputs to a program. Similarly, the intensional database refers to the ground tuples which are derived using the rules in a program, `P`.

Connecting both concepts is the Herbrand base of `P`, written `B(P)`, containing all ground tuples in either the EDB or IDB:

```
B(P) = EDB ∪ IDB(P)
```

### Rules

Rules are written:

```datalog
A :− B1, B2, ..., Bn.
```

Where `A` is an [atom](#atoms) that forms the head of the rule, and `B1, B2, ..., Bn` are [predicates](#predicates) that make up its body. Such a rule can be understood to mean that an instance of `A` should be added to the database if all of the predicates given by `B` through `Z` are true.

The head of a rule may contain variables, in which case a successful application of the rule's body will instantiate them with constants. For example, the following rule projects the `x` attribute of all instances of the `point` relation, to produce matching `xCoordinate` tuples:

```datalog
xCoordinate(x: X) :− point(x: X, y: Y).
```

Such a rule can also be written without reference to any unused attributes:

```datalog
xCoordinate(x: X) :− point(x: X).
```

Rules may also have multiple heads, each with their own body. The following rule is defined using two heads which together compute `zeroPoint` tuples with either their `x` or `y` attribute being `0`:

```datalog
zeroPoint(x: 0, y: Y) :− point(x: 0, y: Y).
zeroPoint(x: X, y: 0) :− point(x: X, y: 0).
```

Rules MUST be range restricted: all variables in the head must appear within the body as part of a [selection predicate](#predicates).

All rules discussed so far are examples of deductive rules. Similarly inductive rules are defined by suffixing a rule's head with `@next`:

```datalog
zeroPoint(x: X, y: Y)@next :− zeroPoint(x: X, y: Y).
```

This causes the rule to delay derived tuples until the next timestep. For more information, see [section 1.2.2](#121-time).

### Predicates

A [rule's](#rules) body may contain a number of different predicates that determine the conditions under which that rule applies.

For example, the following program filters social media profiles to select those with at least 1000 followers:

```datalog
popularProfile(id: X) :−
    profile(id: X),
    c := count : follower(follows: X),
    c >= 1000.
```

Here, `c` is bound to the count of `follower` tuples whose `follows` attribute matches `X`. The result is then constrained to only those variable bindings whose associated counts are greater than 1000.

Detailed descriptions of built-in predicates is provided in [section 1.2.5](#125-predicates).

## 1.2 Semantics

PomoLogic follows a least fixed point semantics for Datalog, with stratified negation and aggregation, and extensions for reifying time and content IDs.

## 1.2.1 Time

Unlike many variants of Datalog, PomoLogic operates over a changing database, and the query engine internally models the progression of time as the database changes. Importantly, this notion of time forms a logical clock which holds no meaning to any other instances of PomoLogic.

The database state is then modeled as a function of time and the program under evaluation, but this computation is intentionally left unspecified, and implementations are free to make their own choices according to their performance constraints and desired features, subject to the following semantics.

First, let `P` be some PomoLogic program, let `next(t)` denote the smallest timestamp `t'`, such that `t < t'`, and let `IDB(t, p)` denote the IDB at time `t`.

Then, for all deductive rules `r` in `P`, if `r` derives a tuple `X` at time `t`, then `X` is in `IDB(t, p)`. Similarly, for all inductive rules `r` in `P`, if `r` derives a tuple `X` at time `t`, then `X` is in `IDB(next(t), P)`.

In order to guarantee a finite execution time for programs containing inductive rules, a few constraints are imposed over their use. In particular, to avoid programs which change infinitely over time for a static EDB, one of two properties must hold for each inductive rule:

1) The head atom appears in the body with the same bindings
2) The body contains at least one positive instantaneous predicate
   1) TODO: Explain what this means!

Treating time in this way is what allows PomoLogic to support both transient tuples—like those that might be used to model events in a user interface—and persistent tuples that capture long-lived data.

For example, given a relation `clicks` whose tuples denote a user clicking on a UI element during the current timestamp, a user interface containing a checkbox might be modeled like so:

```datalog
checkbox(id: Id, state: S)@next     :- checkbox(id: Id, state: S), !clicks(id: Id).
checkbox(id: Id, state: true)@next  :- checkbox(id: Id, state: false), clicks(id: Id).
checkbox(id: Id, state: false)@next :- checkbox(id: Id, state: true), clicks(id: Id).
```

This rule has three heads which together relate the current state of the checkbox to its state in the next timestamp. The first handles the case where no click has occurred, by simply copying the current state forward in time, whereas the next two handle a click event by flipping the checkbox's state at the next timestamp.

Note that while this rule depends negatively on itself, it's able to do so safely because the timestamp of the head is larger than the timestamps of the atoms in the body, stratifying the program with respect to time.

## 1.2.2 Content Addressing

As PomoLogic is intended for use in distributed and decentralized deployments, it is important ensure the use of collision resistant identifiers when referring to tuples. For this purpose, a content addressing scheme is leveraged, wherein facts are associated with a content ID (CID) computed from their structure. The details behind this computation are available in [serialization](./serialization.md).

(Note: this may not actually be the right way of thinking about this, because in many cases, these CIDs are for tuples derived from the EVAC tuples in IPFS, rather than the EVAC tuples themselves. Instead we might also want to accumulate the provenance of variables across joins, and expose a predicate that queries over that provenance. This provenance tracking might even happen conditionally, based on whether it's needed by the program)

This CID is accessed through the selection predicate, which can optionally unify a tuple's CID with a variable:

```datalog
C := point(x: X, y: Y)
```

In this way, these CIDs can be exposed to rules for use in joins between relations, or in the derivation of new tuples.

For example, the following rule operates over an EDB containing tuples which describe the product offerings of an e-commerce site, and computes the number of products belonging to each category:

```datalog
categoryCount(category: Category, count: Count) :-
    Category := category(),
    Count := count : product(category: Category).
```

In this example, the CID of each tuple in the `category` relation acts as its primary key, and is suitable for use as a foreign key when joining against this relation for aggregation purposes.

The choice of CIDs here, rather than more common choices, like auto incrementing IDs or UUIDs, reflects PomoLogic's goals in targeting distributed and decentralized environments, where coordination around the allocation of IDs can't be guaranteed, and where resilience against malicious and byzantine actors is required.

Since content addressing schemes are backed by cryptographically secure hash functions, their use here prevents forgery of IDs by attackers, and guarantees that CID-based dependencies between tuples will be acyclic.

These properties are further leveraged in the design and use of byzantine-fault tolerant CRDTs, as described in [CRDTs](./CRDTs.md).

## 1.2.3 Stratification

PomoLogic supports recursion, negation, and aggregation, and in order to do so safely, it makes use of stratification. Stratification partitions a program's rules into a sequence of strata, based on their dependencies, such that all rules within a stratum depend on each other. These strata are then ordered such that no stratum appears before a stratum containing rules it depends on.

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
4) Append a new stratum to the end, containing all inductive rules. This last stratum corresponds to the modular stratification over time discussed in [section 1.2.1](#121-time), and these rules MAY be implemented using [sinks](#126-sinks)

Such a stratification is only valid if for every strongly connected component in `G`, no edge within that component is labelled with negative polarity. Intuitively, this prevents the use of negation or aggregation through recursive application of rules. Such programs are considered cannot be stratified, and therefore cannot be represented using PomoLogic.

## 1.2.4 Evaluation

Evaluation of PomoLogic proceeds in [timesteps](#121-time), called epochs, which each compute a least fixed point over a batch of changes to the EDB. At each epoch, the program is evaluated in [stratum order](#123-stratification), by evaluating all rules within each stratum to a fixed point before evaluating the next stratum. A fixed point occurs when further applications of a rule against the current EDB and IDB do not result in the derivation of new tuples.

Each epoch is denoted by the timestamp succeeding the last, and begins by evaluating the program's [sources](#125-sources). These act as ingress points for the program, and introduce tuples from the outside world, such as by loading them from a local persistence layer, or by querying them from a remote data source such as IPFS.

Upon evaluating all strata to a fixed point, the program's [sinks](#126-sinks) are evaluated against the current EDB and IDB. These act as egress points for the program, and emit tuples to the outside world for further storage or processing.

When evaluating a stratum, rules MAY be evaluated in any order.

Similarly, when evaluating a rule, predicates MAY be reordered, subject to the following requirements:
- Negation predicates MUST NOT be evaluated until after their atom is fully grounded
- Aggregation predicates MUST NOT be evaluated until any bindings they share with the rule's body are fully grounded
- Constraint predicates MUST NOT be evaluated until both terms are grounded

PomoLogic MAY be implemented over incremental computations, in which case each epoch is RECOMMENDED to operate over deltas of the EDB, wherever possible. A recommended runtime in terms of a [dataflow model](./pomo_flow.md) is provided.

## 1.2.5 Predicates

### Selection

```
Selection ::=
    Atom
  | Var := Atom
```

In both cases, the predicate selects tuples from the relation given by the atom, unifying their attributes against the atom's attributes: unbound variables will be bound to the corresponding value in the tuple, and bound variables will filter the selection to include only tuples with matching values.

The second form additionally unifies the [CID](#122-content-addressing) of the selected tuple with the given variable. If this variable has already been bound, then the selection is filtered to only include the tuple with the bound CID.

### Negation

```
Negation ::= !Atom
```

PomoLogic implements stratified negation, and the negation predicate filters for variable bindings which do not result in any tuples being selected. This predicate MUST be evaluated after all of the variables in the atom's attributes are bound, and it causes the search to fail if it discovers any matching tuples.

For example, the following program computes meal suggestions for pairs of people, such that one person likes the suggestion, and neither dislikes it:

```datalog
suggestedMeal(person1: PersonA, person2: PersonB, meal: Food) :-
  person(name: personA),
  person(name: personB),
  personA != personB,
  likes(name: personA, food: Food),
  !dislikes(name: personB, food: Food)
```

Given an EDB containing:
```datalog
person(name: "Quinn")
person(name: "Brooke")

likes(name: "Quinn", food: "Ramen")
likes(name: "Brooke", food: "Vegan")
likes(name: "Brooke", food: "Schnitzel")

dislikes(name: "Quinn", food: "Vegan")
dislikes(name: "Brooke", food: "Mushrooms")
```

The derived tuples are:

```datalog
suggestedMeal(person1: "Quinn", person2: "Brooke", meal: "Ramen")
suggestedMeal(person1: "Brooke, person2: "Quinn", meal: "Schnitzel")
```

### Aggregation

```
Aggregation ::=
    Var := count : Atom
  | Var := sum Var : Atom
  | Var := min Var : Atom
  | Var := max Var : Atom
  | Var := <UserDefinedAggFun> Var* : Atom
```

PomoLogic's aggregation predicates compute a summary value over a relation for some set of attribute bindings, called the grouping attributes. This summary value is bound to the given variable. If this unification fails, or if the aggregate function returns no result, then the search fails.

Some aggregates, like `sum`, are parameterized over variables which are used to select components of the matched tuples, for aggregation purposes. Such parameters MUST be otherwise unbound, and MUST only appear once within the aggregated atom's attributes.

For example, the following rule is valid:

```datalog
totalStock(total: Total) :- Total := sum Quantity : product(category: Category, quantity: Quantity).
```

But the following is not:

```datalog
totalStock(total: Total) :-
  product(name: Name, category: Category, quantity: Quantity),
  Total := sum Quantity : product(quantity: Quantity).
```

Grouping attributes are used to compute a summary for every possible assignment to the grouping attributes.

For example, the following rule extends the previous valid example to compute the total stock per category of products:

```datalog
totalStock(category: Category, total: Total) :-
  product(category: Category),
  Total := sum Quantity : product(category: Category, quantity: Quantity).
```

Implementations MUST support the aggregate functions defined by [PomoRA](pomo_ra.md#312-group-by), and MAY provide additional non-standard aggregates or allow user defined aggregate functions.

The arguments to an aggregate's atom form a lexical scope under the rule's body, and new bindings introduced in this scope MUST NOT be accessible to other predicates.

For example, the following rule is not range restricted, and thus invalid, because `Category` is unbound in the rule's head:

```datalog
totalStock(category: Category, total: Total) :- Total := sum Quantity product(category: Category, quantity: Quantity).
```

Implementations are RECOMMENDED to implement aggregate functions as linear operations, wherever possible. Such functions can be implemented efficiently in terms of incremental operations over deltas.

### Constraints

```
Constraint ::=
    Term == Term
  | Term != Term
  | Term < Term
  | Term <= Term
  | Term > Term
  | Term >= Term
```

Constraints serve to filter a rule's derived tuples, and their definition follows from their common definitions.

For example, given the following EDB:

```datalog
point(x: 0, y: 0)
point(x: 0, y: 1)
point(x: 0, y: 2)
point(x: 1, y: 0)
point(x: 1, y: 1)
point(x: 1, y: 2)
point(x: 2, y: 0)
point(x: 2, y: 1)
point(x: 2, y: 2)
```

And the following rule:

```datalog
diagonal(x: X, y: Y) :- point(x: X, y: Y), X <= Y.
```

The following tuples are derived:

```datalog
diagonal(x: 0, y: 0)
diagonal(x: 0, y: 1)
diagonal(x: 0, y: 2)
diagonal(x: 1, y: 1)
diagonal(x: 1, y: 2)
diagonal(x: 2, y: 2)
```

## 1.2.5 Sources

Sources introduce tuples from the outside world to a running PomoLogic program. They do so at the beginning of each epoch.

Implementations MAY define their own sources, but sources SHOULD be non-blocking, and are RECOMMENDED to perform any blocking or IO-intensive operations asynchronously.

Sources MAY emit deltas of tuples, if the PomoLogic implementation is able to take advantage of incremental computation.

Implementations MAY also support user defined sources, such as to facilitate the integration of PomoLogic into external systems for persistence or communication.

## 1.2.6 Sinks

Sinks process derived tuples at the end of each epoch.

Implementations MAY define their own sinks, but sinks SHOULD be non-blocking, and are RECOMMENDED to perform any blocking or IO-intensive operations asynchronously.

Implementations MAY also support user defined sinks, such as to facilitate the integration of PomoLogic into external systems for persistence or communication.