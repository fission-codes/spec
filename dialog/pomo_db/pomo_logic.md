# PomoLogic

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Abstract

PomoDB is query language agnostic, and implementations MAY define arbitrary query frontends. This specification describes the semantics for PomoLogic, a variant of Datalog, designed for recursive query processing against PomoDB.

# 1. Introduction

The semantics of PomoLogic match those of Datalog, extended to support queries over time against a dynamic and content-addressable database.

By restricting the expressivity of the engine in this way, it's able to take advantage of the CALM Theorem to ensure the confluence of any programs written using the system.

Datalog is a relational query language that operates on a database of tuples, named atoms, using programmer defined rules, in order to compute a view over that database.

The following section serves as a brief introduction to Datalog, but a comprehensive description of the language and its various extensions is beyond the scope of this specification, and more complete treatments can be found in the [research appendices](../RESEARCH.md).

## 1.1 Notation

This section briefly describes a high-level notation for discussing PomoLogic, however implementations MAY define their own syntax for the language.

### Terms

A Datalog term is either a constant, or a variable. Variables begin with an uppercase character, and constants are written as literals.

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

This causes the rule to delay derived tuples until the next timestep. For more information, see [time](#21-time).

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

For more information, see the detailed descriptions of built-in [predicates](#25-predicates).

# 2. Semantics

PomoLogic follows a least fixed point semantics for Datalog, with stratified negation and aggregation, and extensions for reifying time and content IDs.

## 2.1 Time

PomoLogic imposes additional constraints over PomoDB's concept of [time](../README.md#23-time).

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

## 2.2 Content Addressing

The [CID](../README.md#23-content-addressing) for a tuple is accessed through the selection predicate, which can optionally unify a tuple's CID with a variable:

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

## 2.3 Stratification

TODO: We may want to constrain the use of content identifiers in mutually recursive rules, to avoid the risk of non-terminating queries.

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
4) Append a new stratum to the end, containing all inductive rules. This last stratum corresponds to a [modular stratification over time](#21-time)

Such a stratification is only valid if for every strongly connected component in `G`, no edge within that component is labelled with negative polarity. Intuitively, this prevents the use of negation or aggregation through recursive application of rules. Such programs are considered cannot be stratified, and therefore cannot be represented using PomoLogic.

## 2.4 Evaluation

Evaluation of PomoLogic extends PomoDB's [evaluation semantics](../README.md#27-evaluation) with some additional guarantees.

While the evaluation order for PomoLogic queries is generally unspecified, each epoch MUST be evaluated in [stratum order](#23-stratification), by evaluating all rules within each stratum to a fixed point before evaluating the next stratum. A fixed point occurs when further applications of a rule against the current EDB and IDB do not result in the derivation of new tuples.

When evaluating a stratum, rules MAY be evaluated in any order.

Similarly, when evaluating a rule, predicates MAY be reordered, subject to the following requirements:
- Negation predicates MUST NOT be evaluated until after their atom is fully grounded
- Aggregation predicates MUST NOT be evaluated until any bindings they share with the rule's body are fully grounded
- Constraint predicates MUST NOT be evaluated until both terms are grounded

## 2.5 Predicates

### 2.5.1 Selection

```
Selection ::=
    Atom
  | Var := Atom
```

In both cases, the predicate selects tuples from the relation given by the atom, unifying their attributes against the atom's attributes: unbound variables will be bound to the corresponding value in the tuple, and bound variables will filter the selection to include only tuples with matching values.

The second form additionally unifies the [CID](#22-content-addressing) of the selected tuple with the given variable. If this variable has already been bound, then the selection is filtered to only include the tuple with the bound CID.

### 2.5.2 Negation

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

### 2.5.3 Aggregation

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

Implementations MUST support the aggregate functions defined by [PomoRA](pomo_ra.md#232-group-by), and MAY provide additional non-standard aggregates or allow user defined aggregate functions.

The arguments to an aggregate's atom form a lexical scope under the rule's body, and new bindings introduced in this scope MUST NOT be accessible to other predicates.

For example, the following rule is not range restricted, and thus invalid, because `Category` is unbound in the rule's head:

```datalog
totalStock(category: Category, total: Total) :- Total := sum Quantity product(category: Category, quantity: Quantity).
```

Implementations are RECOMMENDED to implement aggregate functions as linear operations, wherever possible. Such functions can be implemented efficiently in terms of incremental operations over deltas.

### 2.5.4 Constraints

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

# 3. Compilation to PomoRA

TODO: I realize this is very verbose and that it can be simplified and broken up. I'll worry about that after the first pass through describing things :)

The translation to [PomoRA](pomo_ra.md) from PomoLogic is straightforward, and can be performed by first computing a [stratification](#23-stratification) for the program, `S0, S1, ..., Sn`, and then considering each stratum in isolation.

For a given stratum, each [rule is compiled](#31-rule-compilation). If the stratum is recursive, the resulting statements are wrapped in a [loop](pomo_ra.md#loops).

## 3.1 Rule Compilation

Every rule is compiled to an [assignment](pomo_ra#assignment). The rule's head relation appears on the left of the assignment, and a PomoRA expression appears on the right.

This expression is constructed from a query plan given by the terms within the rule's body. Such query plans are not unique, and implementations MAY define their own approach to query planning, but the semantics of the generated query plan MUST match the query plan generated using the following approach.

1) [Associate each variable with a unique name](#311-variables)
2) [Compile selection predicates](#312-selections)
3) [Compile constraints](#313-constraints)
4) [Compile negations](#314-negations)
5) [Compile aggregates](#315-aggregations)
6) [Perform head unification](#316-head-unification)

The result of this process will be a PomoRA expression that computes the output of a given rule, and that expression can be used as the right side of the rule's corresponding [assignment](pomo_ra.md#assignment).

### 3.1.1 Variables

First, for each unique variable, `var`, used in the rule's body, associate it with a unique name, `name(var)`. This name MUST NOT collide with any attributes on relations referenced by the rule.

For example, given variables `x`, `y`, and `z`:
 - `name(x) => "var0"`
 - `name(y) => "var1"`
 - `name(z) => "var2"`

### 3.1.2 Selections

Next, collect every [selection](#251-selection) predicate in the rule's body into a set, named `selections`.

Now, for each predicate in `selections`, we have two cases:
- `atom(a0: v0, ..., an: vn)`
- `Var := atom(a0: v0, ..., an: vn)`

In both cases, start by generating a [projection](pomo_ra#231-projection) operation against `atom`, for the attributes `a0, ..., an`. In the second case, additionally project against the control attribute [$CID](../README.md#221-cid-attribute). Denote this operation by `source`.

Then, for all attributes `a`, with value `v`, and `v` being a constant, generate a [propositional formula](pomo_ra#22-propositional-formula) made up of the conjunction of all such attributes being compared for equality against their associated value.

If this formula contains any terms, then generate a [selection](pomo_ra#233-selection) against `source`, that filters by this formula, and denote the resulting relation by `source`.

Now, for all attributes `a`, with value `v`, and `v` being a variable, if this set is non-empty, then generate a [rename](pomo_ra#232-rename) operation against `source`, that renames every attribute `a` to `name(v)`. 

If the CID of the relation is being bound to a variable, `var`, additionally rename `$CID` to `name(var)`. Store the resulting relation in a set denoted by `sources`.

Next, fold over the relations in `sources`, using the first relation as the initial accumulator. For each pair of relations considered, `left` and `right`, there are three possibilities:
1) `left` and `right` share no common attributes
   - Return the [cartesian product](pomo_ra#236-cartesian-product) of `left` and `right` as the new accumulator
2) The attributes of `right` are fully contained in the attributes of `left` (or the reverse is true)
    - Return the [semijoin](pomo_ra#2310-semijoin) of `left` and `right` as the new accumulator
3) `left` and `right` share some common attributes
    - Return the [natural join](pomo_ra#237-natural-join) of `left` and `right` as the new accumulator

At the end of this process, the fold will have returned a relation corresponding to the joining of all positive terms in the rule's body. Denote that relation `rule_body`.

### 3.1.3 Constraints

Next, collect every [constraint](#254-constraints) in the rule's body into a set, named `constraints`.

Then, compile `constraints` to a single [propositional formula](pomo_ra#22-propositional-formula), `prop_constraints`, made up of the conjunction of all constraints, such that all variables, `var`, are replaced with `name(var)`.

For example:
 - `[x <= 5] => name(x) <= 5`
 - `[x <= y, y = 5] => name(x) <= name(y) AND name(y) = 5`

 If `prop_constraints` is non-empty, then generate a [selection](pomo_ra#233-selection) against `rule_body` using `prop_constraints` as its formula. Denote the resulting relation as the new `rule_body`.

 ### 3.1.4 Negations

Next, collect every [negation](#252-negation) in the rule's body into a set, named `negations`.

Now, for each predicate in `negations`, `!atom(a0: v0, ..., an: vn)`, start by generating a [projection](pomo_ra#231-projection) operation against `atom`, for the attributes `a0, ..., an`. Denote the resulting relation as `negated_relation`.

Then, for all attributes `a`, with value `v`, and `v` being a constant, generate a [propositional formula](pomo_ra#22-propositional-formula) made up of the conjunction of all such attributes being compared for equality against their associated value. If this formula contains any terms, then generate a [selection](pomo_ra#233-selection) against `negation`, that filters by this formula, and denote the resulting relation by `negated_relation`.

Now, for all attributes `a`, with value `v`, and `v` being a variable, if this set is non-empty, then generate a [rename](pomo_ra#232-rename) operation against `negated_relation`, that renames every attribute `a` to `name(v)`. Store the resulting relation in a set denoted by `negated_relations`.

Next, fold over the relations in `negated_relations`, using `rule_body` as the initial accumulator. For each pair of relations considered, `negated_relation` and `accumulator`, return an [antijoin](pomo_ra#2311-antijoin) between `accumulator` and `negated_relation` as the new accumulator.

At the end of this process, the fold will have returned a relation corresponding to the joining of all positive terms in the rule's body, the selection of any constraints, along with the negation of all negative terms in the rule's body. Denote that relation `rule_body`.

### 3.1.5 Aggregations

Next, collect every [aggregation](#253-aggregation) in the rule's body into a set, named `aggregations`.

TODO: aggregation

### 3.1.6 Head Unification

Next, from the rule's head, `atom(a0: v0, ..., an: vn)`, collect each attribute, `a`, with value `v`, and `v` being a variable, and generate a [rename](pomo_ra#232-rename) operation against `rule_body`, that renames every attribute `name(v)` to `a`. The resulting expression will compute the output for the rule.

TODO: add diagrams + examples for each of these steps