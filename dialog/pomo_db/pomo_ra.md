# PomoRA

## Authors

* [Quinn Wilton](https://github.com/QuinnWilton), [Fission](https://fission.codes)
* [Brooklyn Zelenka](https://github.com/expede), [Fission](https://fission.codes)

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

# Abstract

Relational algebra is the theory underpinning relational databases and the query languages against them. It provides a small core language of relational operators that serve as simple building blocks for more complex forms of query processing.

This specification describes PomoRA, a representation of the relational algebra intended for use as an intermediate representation for PomoDB query languages.

# 1. Introduction

PomoRA is a simple embedding of the relational algebra, extended to support recursive queries.

In this way, PomoRA is able to serve as a compilation target for both queries using a familiar SQL-inspired syntax, and for those written using richer, higher-level languages, like [PomoLogic](pomo_logic.md).

Though implementations MAY directly evaluate PomoRA, it is RECOMMENDED that they instead make use of an efficient runtime designed for incremental evaluation, such as [PomoFlow](pomo_flow.md).

# 1.1 Notation

This section briefly describes a high-level notation for discussing PomoRA, however implementations MAY define their own syntax for the language.

A PomoRA program is a [block](#block).

## Block

Blocks are written as a sequence of statements:

```
S1;
S2;
...
Sn;
```

Where each statement, `S1, ..., Sn`, ` is either an assignment, or it denotes a loop.

## Assignment

Assignment is written:

```
R :- E
```

Where `R` denotes a [relation](../README.md#22-relation), and `E` denotes an [operator](#23-operators) in PomoRA.

## Loops

Loops are written:

TODO: Perhaps make the loop's exports explicit here? That isn't strictly necessary, but it does make the dataflow a bit clearer, and would simplify the translation to PomoFlow, because that information is easier to compute from PomoLogic, where everything is declarative

```
while change do
  B
end
```

Where `B` denotes an arbitrary [block](#block).

## Expressions

TODO: I'm not super clear on whether it's worth separating out the syntax for the expressions from the description of their semantics, because that leads to lots of duplication. I'll think about it.

# 2. Semantics

PomoRA extends the relational algebra to support recursive queries by leveraging an inflationary semantics over a loop construct.

A program is evaluated over a database, which can be modeled as a map from [relation](../README.md#22-relation) identifiers, to relations.

Each statement is evaluated in sequence, modifying the database after each step.

## 2.1 Assignment

Assignment in PomoRA is inflationary: it inserts into the relation on the left the contents of the relation given by the expression on the right.

This behavior is REQUIRED in order to guarantee termination of looping queries.

## 2.2 Loops

Loops evaluate over a subquery until a fixed point is achieved over all of the relations assigned to by the subquery.

## 2.3 Operations

Operations take some number of [relations](../README.md#22-relation) as input, transforming them into an output relation.

| Name                                        | Arity |
| ------------------------------------------- | ----- |
| [Projection](#231-projection)               | 1     |
| [Rename](#232-rename)                       | 1     |
| [Selection](#233-selection)                 | 1     |
| [Union](#234-union)                         | N-ary |
| [Difference](#235-difference)               | 1     |
| [Cartesian Product](#236-cartesian-product) | 2     |
| [Natural Join](#237-natural-join)           | 2     |
| [Theta Join](#238-theta-join)               | 2     |
| [Equijoin](#239-equijoin)                   | 2     |
| [Semijoin](#2310-semijoin)                  | 2     |
| [Antijoin](#2311-antijoin)                  | 2     |
| [Group By](#2312-group-by)                  | 1     |

### 2.1.1 Projection

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

### 2.1.2 Rename

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

### 2.1.3 Selection

A selection is a unary operation which is parameterized over a [propositional formula](#22-propositional-formula), and that returns the tuples in its input relation for which this formula holds.

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

### 2.1.4 Union

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

### 2.1.5 Difference

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

### 2.1.6 Cartesian Product

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

### 2.1.7 Natural Join

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

### 2.1.8 Theta Join

A theta join is a binary operator which is parameterized over a [propositional formula](#22-propositional-formula), and that computes the set of all combinations of tuples in its input relations which satisfy that formula. The relations MUST have disjoint attributes, and the tuples in the resulting set are flattened.

This operation is equivalent to composing a [selection](#233-selection) with a [Cartesian product](#236-cartesian-product), however implementations are RECOMMENDED to implement theta joins as a specialization of these operations, for performance reasons.

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

### 2.1.9 Equijoin

An equijoin is a binary operator which is parameterized over a [propositional formula](#22-propositional-formula), where the [propositional formula](#22-propositional-formula) only makes use of equality. It computes the set of all combinations of tuples in its input relations for which the given attribute names are equal. The relations MUST have disjoint attributes, and the tuples in the resulting set are flattened.

This operation MAY be implemented using [theta join](#238-theta-join), however it is RECOMMENDED that equijoins are specialized, for performance reasons.

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

### 2.1.10 Semijoin

A semijoin is a binary operator which returns all tuples in its first input relation such that there is a tuple in its second relation for which the common attributes of both tuples are equal.

This operation is equivalent to composing a [projection](#231-projection) again the first relation's attributes with a [natural join](#237-natural-join), however implementations are RECOMMENDED to implement semijoins as a specialization of these operations, for performance reasons.

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

### 2.1.11 Antijoin

An antijoin is a binary operator which returns all tuples in its first input relation such that there is no tuple in its second relation for which the common attributes of both tuples are equal.

This operation is equivalent to taking the [difference](#235-difference) of the first relation and the [semijoin](#2310-semijoin) of both, however implementations are RECOMMENDED to implement antijoins as a specialization of these operations, for performance reasons.

### 2.1.12 Group By

Group by is the mechanism by which aggregation is performed. Group by is performed over a relation with respect to a set of grouping attributes, and an aggregate function. The operation returns the set of tuples computed by first grouping the input relation, and then applying the aggregate function to each group.

Some aggregates, like `sum`, are parameterized over an attribute name which defines which attribute of the matched tuples to aggregate over.

Implementations MUST support the following aggregates:
- `count` returns the number of matched tuples
- `sum(x)` returns the sum of the selected attribute, `x`, in the matched tuples
- `min(x)` returns the minimum among the selected attribute, `x`, in the matched tuples
- `max(x)` returns the maximum among the selected attribute, `x`, in the matched tuples

Implementations MAY support user defined aggregates, and such aggregates MUST be deterministic and free of side-effects.

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

## 2.2 Propositional Formula

Selection criterion for operations like [selection](#233-selection) and [theta joins](#238-theta-join) are specified in terms of a small subset of propositional logic.

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