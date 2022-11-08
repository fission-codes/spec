# Relational Algebra

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Abstract

The relational algebra forms a common compilation target for Datalog programs. This specification outlines the semantics of the algebra, and describes its operations in terms useful for further compilation to a model with support for incrementalization and recursive queries.

# 1. Introduction

Relational algebra is the theory underpinning relational databases, and query languages against them. Like Datalog, it operates over relations as input, and produces relations as output. Unlike Datalog, it is unable to directly express recursive queries, and such programs must be implemented in terms of a more expressive model, such as one built on [dataflow](dataflow.md).

In the context of Dialog, the relational algebra serves two purposes:
1) Providing a small core language for further compilation to a runtime with support for recursive queries
2) Serving as a common intermediate representation for both queries written using familiar SQL-inspired DSLs, and for those written using [richer, higher-level languages, like Dialog](../dialog/query-engine.md).

# 2. Concepts

## 2.1 Relation

A relation is a set of tuples, where each component of the tuple is called an attribute, and can be referenced by an attribute name.

A n-ary relation contains n-tuples, which can each be written:

```
(a1: v1, a2: v2, ..., an: vn)
```

Where `a1, a2, ..., an` give the names of each attribute, and `v1, v2, ..., vn` their values.

Each tuple within a relation also has a content identifier (CID), as described in the specification of the [query engine](../dialog/query-engine.md#132-content-addressing). This CID can be accessed through a special control attribute that this document will denote as `$CID`: however implementations are RECOMMENDED to use their type system to differentiate between such attributes.

# 3. Operations

An operator specifies an operation against a stream, and is represented by a node. Linear operators are unary operators which can be computed using only the deltas at the current timestamp, and can be compiled to efficient incremental operations. Similarly, bilinear operations are binary operators which are linear with respect to each argument.

All operators take some number of relations as input, transforming them into an output relation.

| Name                                       | Linearity  | Arity |
| ------------------------------------------ | ---------- | ----- |
| [Projection](#31-projection)               | Linear     | 1     |
| [Rename](#32-rename)                       | Linear     | 1     |
| [Selection](#33-selection)                 | Linear     | 1     |
| [Union](#34-union)                         | Linear     | N-ary |
| [Difference](#35-difference)               | Non-Linear | 1     |
| [Cartesian Product](#36-cartesian-product) | Bilinear   | 2     |
| [Natural Join](#37-natural-join)           | Bilinear   | 2     |
| [Theta Join](#38-theta-join)               | Bilinear   | 2     |
| [Equijoin](#39-equijoin)                   | Bilinear   | 2     |
| [Semijoin](#310-semijoin)                  | Bilinear   | 2     |
| [Antijoin](#311-antijoin)                  | Bilinear   | 2     |
| [Group By](#312-group-by)                  | Varies     | 1     |

## 3.1 Projection

A projection is a unary operation which is parameterized over a set of attribute names, and restricts the tuples in its input relation to the attributes given by these names.

For example:

```
projection([:id, :name], [
    (id: 1, name: "Kōbō Abe", born: 1924),
    (id: 2, name: "Jorge Luis Borges", born: 1899),
    (id: 3, name: "Italo Calvino", born: 1923),
    (id: 4, name: "Umberto Eco", born: 1932)
]) => [
    (id: 1, name: "Kōbō Abe"),
    (id: 2, name: "Jorge Luis Borges"),
    (id: 3, name: "Italo Calvino"),
    (id: 4, name: "Umberto Eco")
]
```

## 3.2 Rename

A rename is a unary operation which is parameterized over a pair of attribute names, `(from, to)`, whose input relation contains an attribute named `from`, but does not contain an attribute named `to`. It returns a relation whose tuples are identical to those in the input, except the attribute given by `from` is renamed to `to`.

For performance reasons, implementations are RECOMMENDED to support fusing such operations together, by allowing multiple attributes to be renamed using a single operation.

For example:

```
rename([foo: :bar], [
    (foo: 1),
    (foo: 2),
    (foo: 3)
]) => [
    (bar: 1),
    (bar: 2),
    (bar: 3)
]
```

## 3.3 Selection

A selection is a unary operation which is parameterized over a [propositional formula](#4-propositional-formula), and that returns the tuples in its input relation for which this formula holds.

For example:

TODO: how should I write propositional formulas?
```
selection(born < 1960, [
    (id: 1, name: "Béla Tarr", born: 1955),
    (id: 2, name: "Andrei Tarkovsky", born: 1932),
    (id: 3, name: "Greta Gerwig", born: 1983),
    (id: 4, name: "Věra Chytilová", born: 1929),
    (id: 5, name: "Darren Aronofsky", born: 1969)
]) => [
    (id: 1, name: "Béla Tarr", born: 1955),
    (id: 2, name: "Andrei Tarkovsky", born: 1932),
    (id: 4, name: "Věra Chytilová", born: 1929),
]
```

## 3.4 Union

A union is an n-ary operation which performs a set union over its input relations. All of the input relations must be union-compatible, by sharing the same set of attribute names.

For example:

```
rgb = [
    (name: "Red"),
    (name: "Green"),
    (name: "Blue")
]

cmy = [
    (name: "Cyan"),
    (name: "Magenta"),
    (name: "Yellow")
]

roygbiv = [
    (name: "Red"),
    (name: "Orange"),
    (name: "Yellow"),
    (name: "Green"),
    (name: "Blue"),
    (name: "Indigo"),
    (name: "Violet")
]

union(rgb, cmy, roygbiv) => [
    (name: "Red"),
    (name: "Green"),
    (name: "Blue"),
    (name: "Cyan"),
    (name: "Magenta"),
    (name: "Yellow"),
    (name: "Orange"),
    (name: "Indigo"),
    (name: "Violet")
]
```

For simplicity, implementations MAY implement union as a binary operator, instead simulating n-ary unions through the composition of unions.

For example, using the above relations:

```
union(rgb, cmy, roygbiv) <=> union(rgb, union(cmy, roygbiv))
                         <=> union(union(rgb, cmy), roygbiv)
                         <=> ...
```

## 3.5 Difference

Difference is a binary operation which performs a set difference over its input relations. 

All of the input relations must be union-compatible, by sharing the same set of attribute names.

For example:

```
roygbiv = [
    (name: "Red"),
    (name: "Orange"),
    (name: "Yellow"),
    (name: "Green"),
    (name: "Blue"),
    (name: "Indigo"),
    (name: "Violet")
]

rgb = [
    (name: "Red"),
    (name: "Green"),
    (name: "Blue")
]

difference(roygbiv, rgb) => [
    (name: "Orange"),
    (name: "Yellow"),
    (name: "Green"),
    (name: "Indigo"),
    (name: "Violet")
]
```

## 3.6 Cartesian Product

A Cartesian product is a binary operation which computes all possible pairs between two input relations. The relations MUST have disjoint attributes, and the tuples in the resulting set are flattened.

For example:

```
ranks = [
    (rank: "ace"),
    (rank: "king"),
    (rank: "queen"),
    (rank: "jack")
]

suits = [
    (suit: "spades"),
    (suit: "hearts"),
    (suit: "diamonds"),
    (suit: "clubs")
]

cartesian_product(ranks, suits) => [
    (rank: "ace", suit: "spades"),
    (rank: "ace", suit: "hearts"),
    (rank: "ace", suit: "diamonds"),
    (rank: "ace", suit: "clubs"),
    (rank: "king", suit: "spades"),
    (rank: "king", suit: "hearts"),
    (rank: "king", suit: "diamonds"),
    (rank: "king", suit: "clubs"),
    (rank: "queen", suit: "spades"),
    (rank: "queen", suit: "hearts"),
    (rank: "queen", suit: "diamonds"),
    (rank: "queen", suit: "clubs"),
    (rank: "jack", suit: "spades"),
    (rank: "jack", suit: "hearts"),
    (rank: "jack", suit: "diamonds"),
    (rank: "jack", suit: "clubs")
]

```

## 3.7 Natural Join

A natural join is a binary operator which computes the set of all combinations of the tuples in its input relations, such that the values of any attributes common to both relations are equal.

For example:

```
languages = [
    (language_name: "Ruby", language_appeared: 1995),
    (language_name: "Prolog", language_appeared: 1972),
    (language_name: "Haskell", language_appeared: 1990)
]

web_frameworks = [
    (framework_name: "Rails", language_name: "Ruby"),
    (framework_name: "Sinatra", language_name: "Ruby"),
    (framework_name "weblog", language_name: "Prolog),
    (framework_name: "Yesod", language_name: "Haskell")
]

natural_join(languages, web_frameworks) => [
    (language_name: "Ruby", language_appeared: 1995, framework_name: "Rails"),
    (language_name: "Ruby", language_appeared: 1995, framework_name: "Sinatra"),
    (language_name: "Prolog", language_appeared: 1972, framework_name: "weblog"),
    (language_name: "Ruby", language_appeared: 1990, framework_name: "Yesod"),
]
```

## 3.8 Theta Join

A theta join is a binary operator which is parameterized over a [propositional formula](#4-propositional-formula), and that computes the set of all combinations of tuples in its input relations which satisfy that formula. The relations MUST have disjoint attributes, and the tuples in the resulting set are flattened.

This operation is equivalent to composing a [selection](#33-selection) with a [Cartesian product](#36-cartesian-product), however implementations are RECOMMENDED to implement theta joins as a specialization of these operations, for performance reasons.

For example:

```
budgets = [
    (budget_category_id: 1, budget 100),
    (budget_category_id: 2, budget: 50)
]

products = [
    (product_id: 1, product_category_id: 1, cost: 80),
    (product_id: 2, product_category_id: 1, cost: 90),
    (product_id: 3, product_category_id: 1, cost: 120),
    (product_id: 4, product_category_id: 2, cost: 60),
    (product_id: 5, product_category_id: 2, cost: 20)
]

theta_join(budget_category_id = product_category_id and cost < budget, products) => [
    (product_id: 1, product_category_id: 1, product_category_id: 1, budget: 100, cost: 80),
    (product_id: 2, product_category_id: 1, product_category_id: 1, budget: 100, cost: 90),
    (product_id: 5, product_category_id: 2, product_category_id: 2, budget: 50, cost: 20),
]
```

## 3.9 Equijoin

An equijoin is a binary operator which is parameterized over a [propositional formula](#4-propositional-formula), where the [propositional formula](#4-propositional-formula) only makes use of equality. It computes the set of all combinations of tuples in its input relations for which the given attribute names are equal. The relations MUST have disjoint attributes, and the tuples in the resulting set are flattened.

This operation MAY be implemented using [theta join](#38-theta-join), however it is RECOMMENDED that equijoins are specialized, for performance reasons.

```
categories = [
    (category_id: 1, category_name: "Electronics"),
    (category_id: 2, category_name: "Books")
]

products = [
    (product_id: 1, product_category_id: 1, product_name: "Tele-Multiplay 6000"),
    (product_id: 2, product_category_id: 1, product_name: "Xerox 8"),
    (product_id: 3, product_category_id: 2, product_name: "Foucault's Pendulum"),
    (product_id: 4, product_category_id: 2, product_name: "100 Years of Solitude")
]

equijoin(category_id = product_category_id, products) => [
    (category_id: 1, category_name: "Electronics", product_id: 1, product_category_id: 1, product_name: "Tele-Multiplay 6000"),
    (category_id: 1, category_name: "Electronics", product_id: 2, product_category_id: 1, product_name: "Xerox 8"),
    (category_id: 2, category_name: "Books", product_id: 3, product_category_id: 2, product_name: "Foucault's Pendulum"),
    (category_id: 2, category_name: "Books", product_id: 4, product_category_id: 2, product_name: "100 Years of Solitude")
]
```

## 3.10 Semijoin

A semijoin is a binary operator which returns all tuples in its first input relation such that there is a tuple in its second relation for which the common attributes of both tuples are equal.

This operation is equivalent to composing a [projection](#31-projection) again the first relation's attributes with a [natural join](#37-natural-join), however implementations are RECOMMENDED to implement semijoins as a specialization of these operations, for performance reasons.

For example:

```
users = [
    (user_id: 1, user_name: "Alice"),
    (user_id: 2, user_name: "Bob"),
    (user_id: 3, user_name: "Mallory")
]

admins = [
    (user_id: 1),
    (user_id: 2),
]

semijoin(users, admins) [
    (user_id: 1, user_name: "Alice"),
    (user_id: 2, user_name: "Bob"),
]
```

## 3.11 Antijoin

An antijoin is a binary operator which returns all tuples in its first input relation such that there is no tuple in its second relation for which the common attributes of both tuples are equal.

This operation is equivalent to taking the [difference](#35-difference) of the first relation and the [semijoin](#310-semijoin) of both, however implementations are RECOMMENDED to implement antijoins as a specialization of these operations, for performance reasons.

## 3.12 Group By

Group by is the mechanism by which aggregation is performed. Group by is performed over a relation with respect to a set of grouping attributes, and an aggregate function. The operation returns the set of tuples computed by first grouping the input relation, and then applying the aggregate function to each group.

Implementations MAY support user defined aggregates, but MUST support the aggregate functions described in the [specification for the query language](query-engine.md#aggregation).

For example:

```
products = [
    (product_id: 1, product_category_id: 1, product_stock: 3),
    (product_id: 2, product_category_id: 1, product_stock: 5),
    (product_id: 3, product_category_id: 2, product_stock: 12),
    (product_id: 4, product_category_id: 2, product_stock: 7)
]

group_by(
    product_category_id,
    total_stock,
    sum(product_stock),
    products
) => [
    (product_category_id: 1, total_stock: 8),
    (product_category_id: 2, total_stock: 19),
]
```

# 4. Propositional Formula

Selection criterion for operations like [selection](#33-selection) and [theta joins](#38-theta-join) are specified in terms of a small subset of propositional logic.

A propositional formula is canonically represented in disjunctive normal form, as a disjunction of conjunctives:

```
(A1 and A2 and ... and An) or ... or (Z1 and Z2 and ... Zn)
```

Each conjunctive term includes binary relations between either attributes or constants, using the relational operators: `<`, `<=`, `=`, `!=`, `>=`, `>`, and these relations MAY also be negated.

For example:

```
formula = (not (region = "Europe")) or (accepted_cookies = true)

users = [
    (id: 1, region: "Europe", accepted_cookies: true),
    (id: 2, region: "Europe", accepted_cookies: false),
    (id: 3, region: "Southeast Asia", accepted_cookies: true),
    (id: 4, region: "South America", accepted_cookies: false)
]

select(formula, users) => [
    (id: 1, region: "Europe", accepted_cookies: true),
    (id: 3, region: "Southeast Asia", accepted_cookies: true),
    (id: 4, region: "South America", accepted_cookies: false)
]
```

# 5. Compilation from Dialog

TODO: I realize this is very verbose and that it can be simplified and broken up. I'll worry about that after the first pass through describing things :)

## 5.1 Non-Recursive Dialog

The translation from non-recursive [Dialog](../dialog/query-engine.md) to a relational algebra query plan is straightforward, and can be performed by first compiling each rule in isolation, and then taking the [union](#34-union) of the query plans for any rules with matching head atoms.

A rule is compiled by constructing a query plan from the terms within the rule's body, with respect to the rule's head. Such query plans are not unique, and implementations MAY define their own approach to query planning, but the semantics of the generated query plan MUST match the query plan generated using the following approach.

First, for each unique variable, `var`, used in the rule's body, associate it with a unique name, `name(var)`. This name MUST NOT collide with any attributes on relations referenced by the rule.

For example, given variables `x`, `y`, and `z`:
 - `name(x) => "var0"`
 - `name(y) => "var1"`
 - `name(z) => "var2"`

Next, partition the predicates in the rule's body into four sets: `selections`, `aggregations`, `negations`, and `constraints`, each containing the terms of the respective predicate type.

Now, for each predicate in `selections`, we have two cases:
- `atom(a0: v0, ..., an: vn)`
- `Var := atom(a0: v0, ..., an: vn)`

In both cases, start by generating a [projection](#31-projection) operation against `atom`, for the attributes `a0, ..., an`. In the second case, additionally project against the control column `$CID`. Denote this operation by `source`.

Then, for all attributes `a`, with value `v`, and `v` being a constant, generate a [propositional formula](#4-propositional-formula) made up of the conjunction of all such attributes being compared for equality against their associated value. If this formula contains any terms, then generate a [selection](#33-selection) against `source`, that filters by this formula, and denote the resulting relation by `source`.

Now, for all attributes `a`, with value `v`, and `v` being a variable, if this set is non-empty, then generate a [rename](#32-rename) operation against `source`, that renames every attribute `a` to `name(v)`. If the CID of the relation is being bound to a variable, `var`, additionally rename `$CID` to `name(var)`. Store the resulting relation in a set denoted by `sources`.

Next, fold over the relations in `sources`, using the first relation as the initial accumulator. For each pair of relations considered, `left` and `right`, there are three possibilities:
1) `left` and `right` share no common attributes
   - Return the [cartesian product](#36-cartesian-product) of `left` and `right` as the new accumulator
2) The attributes of `right` are fully contained in the attributes of `left` (or the reverse is true)
    - Return the [semijoin](#310-semijoin) of `left` and `right` as the new accumulator
3) `left` and `right` share some common attributes
    - Return the [natural join](#37-natural-join) of `left` and `right` as the new accumulator

At the end of this process, the fold will have returned a relation corresponding to the joining of all positive terms in the rule's body. Denote that relation `rule_body`.

Then, compile `constraints` to a single [propositional formula](#4-propositional-formula), `prop_constraints`, made up of the conjunction of all constraints, such that all variables, `var`, are replaced with `name(var)`.

For example:
 - `[x <= 5] => name(x) <= 5`
 - `[x <= y, y = 5] => name(x) <= name(y) AND name(y) = 5`

 If `prop_constraints` is non-empty, then generate a [selection](#33-selection) against `rule_body` using `prop_constraints` as its formula. Denote the resulting relation as the new `rule_body`.

 Now, for each predicate in `negations`, `!atom(a0: v0, ..., an: vn)`, start by generating a [projection](#31-projection) operation against `atom`, for the attributes `a0, ..., an`. Denote the resulting relation as `negated_relation`.

 Then, for all attributes `a`, with value `v`, and `v` being a constant, generate a [propositional formula](#4-propositional-formula) made up of the conjunction of all such attributes being compared for equality against their associated value. If this formula contains any terms, then generate a [selection](#33-selection) against `negation`, that filters by this formula, and denote the resulting relation by `negated_relation`.

 Now, for all attributes `a`, with value `v`, and `v` being a variable, if this set is non-empty, then generate a [rename](#32-rename) operation against `negated_relation`, that renames every attribute `a` to `name(v)`. Store the resulting relation in a set denoted by `negated_relations`.

Next, fold over the relations in `negated_relations`, using `rule_body` as the initial accumulator. For each pair of relations considered, `negated_relation` and `accumulator`, return an [antijoin](#311-antijoin) between `accumulator` and `negated_relation` as the new accumulator.

At the end of this process, the fold will have returned a relation corresponding to the joining of all positive terms in the rule's body, the selection of any constraints, along with the negation of all negative terms in the rule's body. Denote that relation `rule_body`.

TODO: aggregation

Next, from the rule's head, `atom(a0: v0, ..., an: vn)`, collect each attribute, `a`, with value `v`, and `v` being a variable, and generate a [rename](#32-rename) operation against `rule_body`, that renames every attribute `name(v)` to `a`. The resulting relation will contain the output for the rule.

Now, taking the [union](#34-union) of all rules which share a head relation will give the query plan for that relation.

Lastly, [stratify](../dialog/query-engine.md#133-stratification) each of these relations and partition the query plans by the strata they belong to.

TODO: add diagrams + examples for each of these steps