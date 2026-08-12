---
title: The Phoenix Architecture
description: Regenerating disposable implementations from durable intent and evaluations
tags:
  - software-architecture
  - artificial-intelligence
  - software-design
---

Chad Fowler's Phoenix architecture is a proposal for software whose
implementations can be regenerated without losing the knowledge that makes the
system work. Fowler argues that generative AI has made code cheap enough to
replace. Behavior, constraints, and evaluation should become the durable
record.[^1]

Fowler borrowed the name from his 2013 argument for immutable servers: replace
a running server with a known-good image instead of repairing it in place.[^2]
Phoenix extends that logic to application code. The code can change as long as
the knowledge needed to judge its replacement survives.

## Systems Outlive Code

Fowler treats observable behavior, interfaces, data, invariants, constraints,
and operational expectations as the durable asset. Code is one expression of
that intent.[^3]

Regeneration happens one component at a time behind an interface the rest of
the system can depend on. The component's inputs, outputs, errors, and side
effects must remain consistent while its internals change. This limits the
blast radius and gives evaluations a defined target.

Fowler's **deletion test** exposes missing knowledge. The test asks what evidence
would establish that a replacement is correct after deleting a component.[^4]
If the old implementation is the only reliable evidence, then its requirements,
edge cases, and failure behavior have not been recorded elsewhere.

The information that must outlive an implementation includes:

- rules for inputs, outputs, errors, and side effects
- business rules and invariants
- data schemas and migration constraints
- architectural decisions, including rejected alternatives
- tests and evaluations that are independent of the current code
- production evidence, such as latency, reliability, and resource limits

An inherited test suite may preserve the implementation's blind spots. Fowler
moves rigor into evaluation: deterministic checks surround probabilistic
generation, and failures surface immediately.[^5]

## Provenance Replaces Diffs

Traditional version control records differences between code snapshots.
Regeneration must also record why generated output exists—which requirement,
constraint, or decision produced it. Fowler calls this **generative provenance**.[^6]

His reference implementation sketches a compiler-like pipeline:

1. The tool divides a plain-language specification into canonical requirements.
2. Requirements and architectural constraints produce implementation units
   with named dependencies and interface rules.
3. A generator produces code for those units.
4. Evaluations and operational evidence determine whether the result is
   acceptable.
5. A change to the intent or a drifted implementation regenerates only the
   affected units.

Phoenix VCS stores a versioned graph of intent, decisions, dependencies,
generated outputs, and evidence. A prompt alone cannot preserve that causal
history. Fowler's repository is a work in progress that tests this model.[^7]

## Conservation

Regeneration requires deliberate stability. Internal service logic may be
cheap to replace, while public APIs, data formats, and user interfaces
accumulate external dependencies and human habits.

Fowler describes the UI as a **conservation layer**. Users learn locations,
workflows, and visual cues; changing them can destroy knowledge that tests do
not capture.[^8] Fowler recommends additive changes, compatibility periods, and
a way back for these slower layers. They provide continuity while teams replace
faster-moving code behind them.

## Application

Phoenix gives teams a method for making one part of an existing system
replaceable:

1. Choose a component with an interface the rest of the system can keep using.
2. Extract its behavior, invariants, data obligations, and operational limits
   from the existing code and production system.
3. Turn those expectations into code-independent API and schema rules, tests,
   and evaluations.
4. Generate or rewrite the internals without changing the component's
   interface.
5. Compare the old and new behavior, roll out gradually, monitor production
   limits, and keep a rollback path.

Phoenix transfers effort from producing code to recording intent and judging
results. Incomplete specifications, weak evaluations, and nondeterministic
generators remain failure modes. Old code also contains decisions no one
recorded. Teams must recover that knowledge and build evaluations strong enough
to reject a plausible but incorrect implementation.

[^1]: Chad Fowler, [“Regenerative Software”](https://aicoding.leaflet.pub/3majnyfydzs2y).

[^2]: Chad Fowler, [“Trash Your Servers and Burn Your Code”](https://chadfowler.com/articles/trash-your-servers-and-burn-your-code.html).

[^3]: Chad Fowler, [“The System Is the Asset”](https://aicoding.leaflet.pub/3mbp5ukeuzs22).

[^4]: Chad Fowler, [“The Deletion Test”](https://aicoding.leaflet.pub/3md5ftetaes2e).

[^5]: Chad Fowler, [“Relocating Rigor”](https://aicoding.leaflet.pub/3mbrvhyye4k2e).

[^6]: Chad Fowler, [“Provenance Is the New Version Control”](https://aicoding.leaflet.pub/3mcbiyal7jc2y).

[^7]: Chad Fowler, [Phoenix VCS](https://tangled.org/chadfowler.com/phoenix/).

[^8]: Chad Fowler, [“UI Is a Conservation Layer”](https://aicoding.leaflet.pub/3mcxo5ojob22c).
