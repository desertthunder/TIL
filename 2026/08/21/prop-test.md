---
title: Property-Based Testing
description: Generating and shrinking inputs that challenge general claims about code
tags:
  - testing
  - software-design
  - quality-assurance
---

An example-based test checks behavior at values chosen by the programmer. A
property-based test states a claim over a range of values, generates many examples from
that range, and searches for a counterexample. QuickCheck popularized this approach for
Haskell in 2000. Libraries such as Hypothesis, jqwik, ScalaCheck, fast-check, and proptest
apply it in other languages.[^1]

Sampling cannot prove that a property holds for every input but it can explore a much larger
and more varied part of the input space than a short hand-written list, especially when
the generator understands the structure of valid inputs.

A property-based test has four working parts:

1. A **property** says what must hold.
2. A **generator** or strategy produces inputs in the property's domain.
3. The runner searches that domain, usually combining pseudo-random choices with known
   edge cases and previously failing examples.
4. A **shrinker** reduces a failure to a smaller counterexample that still fails.

The property is the test oracle. The generator determines what evidence the test will
actually see. Weakness in either can leave large classes of bugs untouched.

## Properties

Properties often describe relationships rather than exact outputs:

- **Invariant:** sorting preserves length and element multiplicity.
- **Round trip:** decoding an encoded value returns the original value.
- **Idempotence:** normalizing an already normalized value changes nothing.
- **Metamorphic relation:** translating every point in a geometry problem translates the
  answer by the same amount.
- **Differential oracle:** a new implementation agrees with a slower reference
  implementation.
- **Model agreement:** each operation on a stateful system produces the state predicted
  by a simpler model.

These patterns are starting points, not automatic specifications. `decode(encode(x)) == x`
checks one direction of a codec, but says nothing about rejecting malformed encodings or
whether `encode(decode(bytes))` preserves a non-canonical representation. A sort test that
checks only ordering can pass after dropping duplicate elements.

A small example with Hypothesis expresses both necessary sorting properties:

```python
from collections import Counter

from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort(values):
    result = my_sort(values)
    assert all(a <= b for a, b in zip(result, result[1:]))
    assert Counter(result) == Counter(values)
```

Hypothesis supplies lists of different lengths and integers including boundary-like
values, then shrinks any failure toward a simpler list.[^2]

## Generators

Generate values from the real input domain.

- An abstract syntax tree generator should construct valid trees recursively.
- A request generator should respect relationships between fields.
- A state-machine generator should choose only operations valid in its
  current state.

Composing generators usually gives the shrinker more structure than
creating arbitrary bytes and rejecting almost all of them.

Use filtering or assumptions for conditions that are cheap and frequently true. If a
test generates arbitrary integer pairs and rejects every pair except those satisfying a
rare relation, most of its budget is spent producing no evidence. Construct related
values directly instead.

Distribution matters even when every generated value is valid. Uniformly selecting a
64-bit integer almost never produces `0`, `1`, or an overflow boundary by chance. Good
libraries deliberately include simple and exceptional values, but application-specific
boundaries still belong in the strategy. Classify generated cases by size, shape, feature,
or code path so a green test does not hide a search concentrated on trivial inputs.

Keep generators no broader than the declared property. If a function accepts only valid
UTF-8, invalid byte strings belong in a separate rejection property. Mixing valid and
invalid domains tends to fill a test with preconditions and makes failures harder to
interpret.

## Shrinking and Replay

The first failing input may be large because it was found after many generation choices.
Shrinking repeatedly tries simpler related inputs while preserving the failure. Depending
on the type, that may mean moving numbers toward zero, deleting list elements, shortening
strings, or removing operations from a state-machine trace. Hypothesis strategies define
type-specific shrink behavior. Its integer strategy, for example, shrinks toward zero.[^3]

“Minimal” means minimal according to the library's shrink order, not necessarily the
smallest mathematical counterexample. Custom generators should preserve enough structure
for useful shrinking. Excessive filtering can block a shrink path because a simpler
candidate no longer satisfies the assumptions.

A failing counterexample must be reproducible. Record the seed or serialized case when
the framework requires it. Hypothesis stores failures in an example database and replays
them on later runs. For permanent regression coverage, its documentation recommends
adding the concrete value as an explicit example rather than relying on the database's
internal representation.[^4]

## State

A single generated value is insufficient for caches, editors, databases, protocols, and
other systems whose behavior depends on operation history. Stateful property-based tests
generate sequences such as create, update, delete, undo, and query. After each step they
check invariants or compare the implementation with a small in-memory model.

Hypothesis's rule-based state machines generate both action sequences and their arguments,
then shrink a failure into a short program. Its documentation demonstrates a database
compared with a dictionary-and-set model. The failing trace can shrink to creating a key
and value, saving it, deleting it, and observing a disagreement.[^5]

Concurrency needs more than random values. The generated input may need to include a
schedule, message delivery order, failure point, or clock advance. Making nondeterminism
an explicit generated value allows the runner to replay and shrink it.

## Usage

Property-based tests complement example-based tests. Keep examples for named regressions,
important scenarios, and behavior whose expected output is clearest as a literal value.
Use generated tests where a rule spans a combinatorial domain or a sequence of actions.

Start with pure functions and round trips, then add reference models or state machines
where the extra machinery pays for itself. Run a moderate deterministic profile in the
normal test suite and a larger search in scheduled CI. When a property fails, decide
whether the implementation, property, or generator is wrong. Finding an unstated domain
condition is also useful design feedback.

[^1]: Koen Claessen and John Hughes, [“QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs”](https://doi.org/10.1145/351240.351266), 2000.

[^2]: Hypothesis, [documentation and introductory example](https://hypothesis.readthedocs.io/en/latest/).

[^3]: Hypothesis, [strategies reference](https://hypothesis.readthedocs.io/en/latest/reference/strategies.html).

[^4]: Hypothesis, [“Replaying failed tests”](https://hypothesis.readthedocs.io/en/latest/tutorial/replaying-failures.html).

[^5]: Hypothesis, [“Stateful tests”](https://hypothesis.readthedocs.io/en/latest/stateful.html).
