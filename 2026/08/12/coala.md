---
title: Cognitive Architectures for Language Agents
description: A framework for reasoning about an agent's memory, actions, and decision loop
tags:
  - artificial-intelligence
  - agents
  - cognitive-architecture
  - memory
---

A language model maps text to text. A language agent wraps that model in a program that
can retain state, choose among actions, and respond to the results.

Cognitive Architectures for Language Agents (CoALA) supplies a vocabulary for describing
an agent's components[^1] and leaves implementation to system designers.

Theodore Sumers, Shunyu Yao, Karthik Narasimhan, and Thomas Griffiths separate agents
into three concerns:

- memory, split into working memory and several forms of long-term memory
- an action space containing internal and external actions
- a decision procedure that plans, selects, and executes actions in a loop

## From productions to language agents

CoALA draws on production systems and cognitive architectures from symbolic AI. A
production system repeatedly applies rules that transform symbolic state. A cognitive
architecture adds the memories, control flow, and interfaces needed to turn those rules
into an agent.

The authors treat an LLM as a stochastic counterpart to a production system. Instead of
applying a hand-written rule to a string, it defines a distribution over possible textual
continuations or transformations. Prompt templates, parsers, retrieval functions, and
ordinary program code constrain those transformations and connect them to stored state
and an environment.[^1]

This analogy shifts the unit of design from the model to the whole agent. Model parameters
supply broad, implicit procedural knowledge. Code supplies deterministic procedures and
control flow. Designers can then decide which work needs the model's flexibility and which
belongs in inspectable code.

## Memory

An LLM call is stateless, even if an application passes earlier messages back on the next
call. A CoALA agent maintains explicit memory across calls. Working memory is the hub: the
program assembles part of it into a prompt, parses the model's response back into variables,
and uses those variables to invoke other procedures.

| Memory     | What it contains                                                | Examples                                     |
| ---------- | --------------------------------------------------------------- | -------------------------------------------- |
| Working    | Active state for the current decision cycle                     | observations, goals, retrieved facts, a plan |
| Episodic   | Records of the agent's experience                               | prior interactions, trajectories, failures   |
| Semantic   | Knowledge about the world and the agent                         | documents, facts, maps, learned conclusions  |
| Procedural | Knowledge of how to reason, decide, learn, and affect the world | LLM weights, prompts, functions, tool code   |

Working memory is broader than the model's context window because it's an application data
structure that persists across model calls, while each prompt contains only a selected
view of that structure.

Procedural memory has an important split between weights & source code. The model's
weights contain implicit procedures, while the agent's source code contains explicit ones.
That code includes implementations of actions as well as the top-level decision loop.
Episodic and semantic memories may be empty, but a designer must supply enough
procedural memory to bootstrap the agent.

## Actions

CoALA classifies an action by what it reads or changes:

| Action    | Operation                                        | Result                        |
| --------- | ------------------------------------------------ | ----------------------------- |
| Reasoning | Read and update working memory with the LLM      | working memory to itself      |
| Retrieval | Read long-term memory into working memory        | long-term to working memory   |
| Learning  | Write working-memory content to long-term memory | working to long-term memory   |
| Grounding | Act on an environment and receive observations   | agent to external environment |

Reasoning can summarize an observation, reflect on a failed attempt, generate a plan, or
evaluate candidate actions. It produces temporary state unless a later learning action
commits its result to long-term memory.

Retrieval can read any long-term memory. It might recall a similar episode, load factual
knowledge, or find an executable skill. The retrieval procedure may use rules, keyword
search, embeddings, model-based scoring, or a combination of them.

Learning covers every write to long-term memory, including experiences, distilled facts,
prompts, skills, and parameter updates. An agent that writes code can adapt its own
procedures, but it can also introduce bugs or defeat the designer's constraints.[^1]

Grounding covers interaction with physical, social, and digital environments. Robot
control, dialogue, browser use, API calls, and code execution are all grounding actions
when they affect systems outside the agent. Observations return through a grounding
interface and enter working memory.

## Decision Loop

The decision procedure is the agent's `main` loop. A cycle has two stages:

1. During planning, reasoning and retrieval propose one or more grounding or learning
   actions, evaluate them, and select one. Proposal and evaluation may repeat.
2. During execution, the program invokes the selected procedure. Its result changes
   long-term memory or the external environment, a new observation enters working memory,
   and the next cycle begins.

The framework therefore distinguishes internal actions used to plan from the action that
the cycle ultimately commits. A simple agent may generate one external action immediately.
A deliberative agent can generate several candidates, simulate or score their consequences,
reject all of them, and try again. Tree search and classical planning algorithms can
provide that outer control structure while an LLM proposes or evaluates states.

Allowing an agent to choose when to learn changes the decision problem. Many systems write
on a fixed schedule, such as saving a summary after every task. CoALA makes that write a
selectable action, so the agent may decide whether an observation is worth retaining and
which memory should receive it. The agent then trades progress on the current task against
better behavior on future tasks.

## Existing CoALA Agents

CoALA gives us more precise terms for systems that are all loosely called "agents." The
paper classifies five examples:[^1]

| System            | Memory and internal actions                                  | Decision pattern                                            |
| ----------------- | ------------------------------------------------------------ | ----------------------------------------------------------- |
| SayCan            | Procedural memory only; no reasoning, retrieval, or learning | Scores a fixed set of robot skills and executes one         |
| ReAct             | Working and procedural memory; reasoning                     | Alternates one reasoning step with one external action      |
| Voyager           | Retrieves and writes executable skills                       | Proposes tasks, writes code, tests it, and stores successes |
| Generative Agents | Episodic and semantic memory; retrieval, reasoning, learning | Retrieves experiences, reflects, plans, and acts            |
| Tree of Thoughts  | Working memory and reasoning; no long-term memory            | Searches and evaluates branches before submitting an answer |

ReAct's defining addition is an internal reasoning action inside an environment loop.
Voyager and Generative Agents add retrieval and learning, though they write different
memories. Tree of Thoughts has elaborate decision-making despite almost no external action
space. The authors maintain a broader catalog using the same categories.[^2]

## Lessons

Using CoALA, a designer starts with action and memory boundaries before polishing prompts.

- Give the agent only the long-term memory modules it needs. A customer assistant may write its
  interaction history but should not be able to edit inventory facts or its own code.
- Define read and write permissions separately. Retrieval from semantic memory does not
  imply permission to change it.
- Keep the action space as small as the task permits. More actions increase both capability
  and the difficulty of choosing safely among them.
- Use code for generic algorithms and hard constraints, and use the LLM where flexible
  language interpretation or generation is valuable.
- Budget deliberation. Additional model calls can improve a plan, but they add latency and
  compute. An agent needs a stopping rule or a way to estimate whether more thought helps.
- Evaluate the action space for worst cases. Grounding can harm external systems, while
  learning can corrupt the agent's memory or procedures.

The boundary between the agent and its environment depends on control and coupling. A
public Wikipedia is an external environment because others can change it. An offline copy
controlled by the agent can be semantic memory. Code execution in an isolated internal
simulator may be reasoning. Execution on another machine is grounding. A consistent
boundary makes permissions and failure modes easier to identify.

## Limitations

CoALA offers a taxonomy and design agenda. The paper does not implement a standard runtime,
run experiments showing that the architecture improves performance, or establish that its
memory categories reproduce human cognition. Its examples organize prominent systems
available in 2023 rather than proving that every language agent must use these modules.

The framework also leaves the hard mechanisms open. It names retrieval without deciding
what should be recalled, learning without deciding what deserves persistence, and planning
without supplying a reliable evaluator. The authors identify calibration, hallucinated
self-evaluation, alignment, deletion and correction of memories, and safe modification of
agent code as open problems.

Before implementation, a designer still has to answer four questions:

1. What state persists?
2. Which procedures may read or change it?
3. What can affect the outside world?
4. Which code chooses among those operations?

Stronger or multimodal models may absorb some explicit mechanisms, but the host program
still controls persistence and access to the environment.

[^1]: Theodore R. Sumers, Shunyu Yao, Karthik Narasimhan, and Thomas L. Griffiths, ["Cognitive Architectures for Language Agents," _Transactions on Machine Learning Research_, 2024](https://arxiv.org/html/2309.02427v3).

[^2]: Shunyu Yao et al., ["CoALA: Awesome Language Agents"](https://github.com/ysymyth/awesome-language-agents), an author-maintained catalog organized with the CoALA framework.
