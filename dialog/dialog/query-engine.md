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

$$ x(x_1, x_2, ..., x_n) $$

Where $x$ is the name of the atom, and $x_1, x_2, ..., x_n$ are its arguments. The arguments to an atom can be any [term](#terms), and an atom whose arguments are all constants is called a ground atom.

For example, $\operatorname{point}(3, 7)$ is a ground atom named $\operatorname{point}$ with two arguments, $3$ and $7$. Similarly, $\operatorname{point}(X, Y)$ is a non-ground atom with two variable arguments, $X$ and $Y$.

### Rules

Rules are written:

$$ A \coloneq B_1, B_2, ..., B_n. $$

Where $A$ is an [atom](#atoms) that forms the head of the rule, and $B_1, B_2, ..., B_n.$ are [predicates](#predicates) that make up its body. Such a rule can be understood to mean that an instance of $A$ should be added to the database if all of the predicates given by $B_1$ through $B_n$ are true.

The head of a rule may contain variables, in which case a successful application of the rule's body will be used to instantiate them with constants. For example, the following rule projects the first argument of the $\operatorname{point}$ atom, to produce matching instances of the $\operatorname{xCoordinate}$ atom:

$$ \operatorname{xCoordinate}(X) \coloneq \operatorname{point}(X, Y). $$

Such a rule can also be written using wildcards in the place of unused variables:

$$ \operatorname{xCoordinate}(X) \coloneq \operatorname{point}(X, \_). $$

Rules may also have multiple clauses, each with their own head and body. The following rule is defined using two clauses which together compute instances of $\operatorname{zeroPoint}$ where either the first or the second argument is $0$:

$$
\begin{align*}
\operatorname{zeroPoint}(0, Y) &\coloneq \operatorname{point}(0, Y). \cr
\operatorname{zeroPoint}(X, 0) &\coloneq \operatorname{point}(X, 0). \cr
\end{align*}
$$

An inductive rule is defined by suffixing the rule's head with $@\operatorname{next}$:

$$
\begin{align*}
\operatorname{zeroPoint}(X, Y)@\operatorname{next} &\coloneq \operatorname{zeroPoint}(X, Y). \cr
\end{align*}
$$

This causes the rule to propagate tuples for the current timestamp forward to the next timestep. For more information, see [section 1.3.2](#131-time).

### Predicates

A [rule's](#rules) body may contain a number of different predicates that determine the conditions under which that rule applies:

$$
\begin{align*}
Predicate &\Coloneqq Selection \cr
  &\mathrel{|} Negation \cr
  &\mathrel{|} Constraint \cr
  &\mathrel{|} Var \coloneqq Aggregation \cr
\cr
Selection &\Coloneqq Atom \cr
  &\mathrel{|} Var \coloneqq Atom \cr
  &\mathrel{|} Var \coloneqq Atom \mathrel{@} Var \cr
  &\mathrel{|} Atom \mathrel{@} Var \cr
\cr
Negation &\Coloneqq \neg Atom \cr
\cr
Aggregation &\Coloneqq \operatorname{count} : Atom \cr
  &\mathrel{|} \operatorname{sum} Var : Atom \cr
  &\mathrel{|} \operatorname{min} Var : Atom \cr
  &\mathrel{|} \operatorname{max} Var : Atom \cr
  &\mathrel{|} \operatorname{AggFun} Var\ast : Atom \cr
\cr
Constraint &\Coloneqq Term = Term \cr
  &\mathrel{|} Term \ne Term \cr
  &\mathrel{|} Term \lt Term \cr
  &\mathrel{|} Term \le Term \cr
  &\mathrel{|} Term \gt Term \cr
  &\mathrel{|} Term \ge Term
\end{align*}
$$

For example, the following program filters social media profiles to select those with at least 1000 followers:

$$
\begin{align*}
\operatorname{popularProfile}(X) \coloneq &\operatorname{profile}(X), \cr
    & c \coloneqq \operatorname{count} : \operatorname{follower}(X, \_), \cr
    &c \ge 1000.
\end{align*}
$$

Here, $c$ is bound to the count of instances of $\operatorname{follower}$ whose first argument matches $X$. The result is then constrained to those counts which are greater than 1000.

Detailed descriptions of these predicates is provided in [section 1.3](#13-semantics).

## 1.2 Types

Dialog is designed to be run within WebAssembly, and so its types and their encodings are informed by the format. See the [WebAssembly specification](https://webassembly.github.io/spec/core/appendix/index-types.html) for more information.

Note, however, that only some types are supported as Dialog primitives. These are:

- [Numbers](https://webassembly.github.io/spec/core/syntax/types.html#syntax-numtype)
- [Opaque Reference Types](https://webassembly.github.io/spec/core/syntax/types.html#reference-types)

As WebAssembly does not define common types such as booleans or strings, these are handled using opaque reference types, and more information is available in the [serialization](./serialization.md) specification.

## 1.3 Semantics

Dialog follows a fixed point semantics for Datalog, with stratified negation and aggregation, and extensions for reifying time and content IDs.

## 1.3.1 Time

## 1.3.2 Evaluation