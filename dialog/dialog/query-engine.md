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

### Rules

Rules are written:

```datalog
A :− B1, B2, ..., Bn.
```

Where `A` is an [atom](#atoms) that forms the head of the rule, and `B1, B2, ..., Bn` are [predicates](#predicates) that make up its body. Such a rule can be understood to mean that an instance of `A` should be added to the database if all of the predicates given by `B` through `Z` are true.

The head of a rule may contain variables, in which case a successful application of the rule's body will be used to instantiate them with constants. For example, the following rule projects the first argument of all instances of `point`, to produce matching instances of `xCoordinate`:

```datalog
xCoordinate(X) :− point(X, Y).
```

Such a rule can also be written using wildcards in the place of unused variables:

```datalog
xCoordinate(X) :− point(X, _).
```

Rules may also have multiple clauses, each with their own head and body. The following rule is defined using two clauses which together compute instances of `zeroPoint` where either the first or the second argument is `0`:

```datalog
zeroPoint(0, Y) :− point(0, Y).
zeroPoint(X, 0) :− point(X, 0).
```

An inductive rule is defined by suffixing the rule's head with `@next`:

```datalog
zeroPoint(X, Y)@next :− zeroPoint(X, Y).
```

This causes the rule to propagate tuples for the current timestamp forward to the next timestep. For more information, see [section 1.3.2](#131-time).

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
  | Var := Atom @ Var
  | Atom @ Var

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

Here, `c` is bound to the count of instances of `follower` whose first argument matches `X`. The result is then constrained to those counts which are greater than 1000.

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

## 1.3.2 Evaluation