# Gatter v0.1: an IO-free kernel for composed behaviour

Status: draft, 2026-11-26. Nothing is built. A path outside this repository starts with the name
of a repository of the beyond10x organization (`canon/…` is `github.com/beyond10x/canon`).

## 0. Origin

The owner, 2026-11-26 (dictated), in three steps:

1. An LLM pass-through upstream in llm-gateway, harness-agnostic driving by base URL + config,
   then: "some sort of modular pipelining similar to what we do on b10x/inference but just with the
   llm layer … something to build logic gates, electronic switchboard, low level, components have
   contracts about their inputs and outputs, and based on its compatibility various structures can
   become possible … compose all kind of streams, data-pipelines, decisions nodes, logic modules
   (state-machines/…".
2. "from low-level (logic) up to complexer systems like state-machines, behaviour trees, super
   little components, which then compose to more and more higher level abstractions. | this could
   become some sort of behaviour spec - where certain parts are dynamic, others deterministic, then
   this again can be explored in its state space with simulations. if we then could create partial
   clones (structural copies of those, slightly mutated (sparse) - and then evaluated and simulated
   all at the same time".
3. Choosing a new repository: "E … and modelspec - we compose higher-level structures from lower
   level atomic components down to logic or electronic … and some nodes would be trainable on a GPU
   (lowered/lifted modelspec) - io free kernel, just specification, generic, but flexible, meta-spec
   included, eg consumers of the tool/lib can build their own easily by composition or contracts
   (conformance)".

A survey of the existing repositories preceded this design. The findings that matter are restated
below with their paths.

## 1. What Gatter is

Gatter is a specification kernel: a document format and a pure Rust library that describe a
behaviour as a circuit of components with typed contracts, check that the circuit is well formed,
evaluate it tick by tick, explore its state space, and generate and score sparse structural
variants of it. It performs no IO. A host gives it text and values and gets text and values back.

The name: German *Gatter*, a logic gate. German digital-logic terms name the two halves of the
model: a *Schaltnetz* is a combinational circuit (output = f(input)), a *Schaltwerk* a sequential
one (output = f(input, state)). Gatter has both.

Names considered beside it: `schaltwerk` (the sequential-circuit term) and `netlist` (the
electronics term for "components and their connections").

## 2. Goals and non-goals

Goals:

- **G1 IO-free kernel.** No file, socket, clock, environment, process, thread, GPU or randomness
  source inside the kernel. Time is a tick count given by the caller; randomness is a seed given by
  the caller. Every function of the kernel is a function of its arguments.
- **G2 Atoms up to abstractions.** A small fixed set of atomic components (gates, registers,
  state machines, behaviour-tree nodes, oracles, learned nodes); everything else is a composite,
  and a composite is a component.
- **G3 Typed contracts.** Every port has a type; every component has a contract (assumptions on
  its inputs, guarantees on its outputs). A connection is admitted only when the contracts fit.
- **G4 Conformance.** Every atom and every composite carries scenarios (tick traces). Any
  implementation of a component, in the kernel or in a host, is held to them.
- **G5 Meta-spec.** The vocabulary itself (component kinds, type constructors, contract forms) is
  declared in the same format, so a consumer declares its own component kinds by composition or by
  contract + conformance, without changing Gatter.
- **G6 Deterministic and dynamic parts.** A dynamic part (an LLM call, a person, an external API)
  is an *oracle*: declared outcomes, optionally a distribution, never executed by the kernel.
- **G7 Trainable nodes.** A *learned* node has a typed tensor signature and is lowered to a
  modelspec `.mspec` program for training on a GPU by the host; trained weights are lifted back
  into the spec.
- **G8 Simulation and search.** Exhaustive exploration of finite circuits, seeded random walks of
  large ones, and a search over sparse mutated variants evaluated in parallel.

Non-goals:

- Not a runtime with IO. Streams, sockets, queues, clocks and retries are the host's (inference,
  spindle, loom, llm-gateway).
- Not a service specification language. Entities, commands, storage and HTTP are ESS's.
- Not a legitimacy calculus. Evidence, obligations and authority are canon's.
- Not a model compiler. Tensor programs, gradients and GPU kernels are modelspec's.

## 3. The model

### 3.1 Time: synchronous ticks

Gatter uses the synchronous model of hardware and of synchronous languages (Lustre, Esterel,
SCADE): at each tick every input has a value (or `absent`), the combinational part is computed in
topological order, then every sequential element takes its next state. A run is a sequence of
ticks. Every circuit is deterministic given its inputs, its oracle answers and its seed.

- Rule T1: a cycle in the connection graph must pass through a sequential element (a register or
  a state machine). A combinational loop is refused at check time, as in hardware.
- Rule T2: absence is a value. A port that carries nothing at a tick carries `absent`; a stream is
  the sequence of present values. Event streams and sampled signals share one model.
- Asynchrony lives in the host: a host maps its events (an inference envelope, an HTTP request, a
  timer) to ticks.

### 3.2 Types

- Scalars: `bool`, `tri` (True / False / Unknown, Kleene logic, the three values of canon's claims,
  `canon/crates/canon/src/model/predicate.rs:98-104`), bounded `int<lo..hi>`, `enum{...}`,
  `bits<n>`.
- Structured: `record{...}`, `option<T>`, `list<T, max>` (bounded), `text` and `bytes`
  (unbounded; a port of such a type makes a circuit non-finite for exhaustive exploration).
- Tensors: `tensor<dtype, shape>`, modelspec's type (`Tensor<f32|i32, [d..]>`,
  `modelspec/website/docs/concepts/language.md:50-53`), on learned-node ports and the ports that
  feed them.
- Schema: `schema<id>` names a JSON Schema document carried by the spec, for consumer payloads
  (an LLM request, an A2A message). Checked as a type by the subsumption rule of 3.4.

### 3.3 Contracts

A contract is assume/guarantee over ports:

- `assume`: predicates over input values (and over a bounded tick history): what a caller must meet.
- `guarantee`: predicates over output values given inputs and state.
- `state`: the type of the component's internal state, if sequential.
- `latency`: ticks from input to output (0 for combinational).

Predicates are the same small language as canon's (all / any / not over comparisons and
three-valued tests), so a contract over `tri` can be lowered to canon (section 6.3).

### 3.4 Compatibility

A connection from output port `o` of A to input port `i` of B is admitted when:

1. `type(o)` is a subtype of `type(i)`: an int range inside the other, an enum subset, a record
   with every required field, a schema whose instances all validate against the other. For schemas
   the rule is canon's structural narrowing/widening classifier applied to two schemas instead of
   two revisions of one (`canon/crates/canon/src/diff/arguments.rs:1-50`).
2. A's guarantee on `o` implies B's assumption on `i`. Decided where the predicate language makes
   it decidable (finite domains); otherwise recorded as an *obligation* that conformance must show,
   never silently admitted.
3. Fan-out is free. Fan-in needs an explicit `merge` atom (3.5); two outputs on one input are
   refused.

Refusal words follow swarm's declared vocabulary where the meaning is the same (`SchemaMismatch`,
`swarm/src/core/domains/blackbox.yaml:429`).

### 3.5 Atoms

Each atom has its contract and its scenarios in `spec/atoms/` (to be written).

- Combinational (*Schaltnetz*): `and`, `or`, `not`, `xor` (over `bool` and `tri`), `mux`
  (select by an enum), `compare`, `map` (a total function given as a table or a predicate),
  `pack` / `unpack` (records), `merge` (fan-in with a declared priority), `encode` / `decode`
  (enum to and from an `i32` index tensor, the boundary to learned nodes, section 5).
- Sequential (*Schaltwerk*): `register` (delay one tick), `counter`, `timer` (counts ticks; no
  wall clock), `fsm` (Mealy or Moore: states, guarded transitions, outputs).
- Behaviour tree: `sequence`, `fallback`, `parallel(k of n)`, `decorator` (invert, retry n,
  timeout in ticks), `condition`, `action`. Status is `success | failure | running`. A behaviour
  tree compiles to an `fsm` over child statuses, so it needs no evaluation rule of its own.
- `oracle`: a dynamic part (section 4).
- `learned`: a trainable part (section 5).

### 3.6 Composites

A composite is a set of instances, connections and exposed ports, and it is a component: it has
ports and a contract (derived, or declared and checked against the derived one). Composites nest
without limit; a check names the instance path at fault (`router/filter/and#2.in[1]`).

### 3.7 The meta-spec

The format describes its own vocabulary. `spec/meta/` declares the atom kinds, the type
constructors and the contract forms in the same document format. A consumer adds a component kind
in one of two ways:

- **By composition**: a named composite, published with its contract and scenarios. An "LLM
  route selector" is a `mux` over a `map` from request fields to a route enum, plus a `fallback`.
- **By contract**: a new kind with a contract and conformance scenarios and no body. The kernel
  treats it as an opaque atom; the host provides the implementation; the scenarios are the
  conformance test that implementation must pass. ESS components that a host implements and a
  conformance suite checks have the same shape (`a2a/ess/components.yaml`; `ess verify conform`).

### 3.8 Conformance

A scenario is a tick trace: inputs per tick, oracle answers per tick, expected outputs and states.
The kernel runs scenarios against its own evaluation of a composite; a host runs the same scenarios
against its implementation of an opaque kind. A scenario set is checked for strength by mutation
(7.3), as canon does with `canon mutate` (`canon/crates/canon/src/mutate/mod.rs:1-30`) and ESS with
`ess verify conform mutate` (`ess/crates/verify/ess-conformance/src/mutate.rs:1-40`).

## 4. Dynamic parts: oracles

An `oracle` has typed inputs and outputs and a declared outcome set: the idea of canon's
`author: judgement` (`canon/website/docs/concepts/protocols.md:211-260`), ESS's `external:`
outcomes (`ess/docs/design/guarded-external-outcomes.md:1-10`) and spindle's
`external.<command>.<outcome>` capability (`spindle/README.md:121-153`).

- The kernel never computes an oracle's output. In evaluation the host passes it per tick; in a
  scenario the trace carries it; in exploration the explorer enumerates or samples the outcomes.
- An oracle may declare a distribution over its outcomes and a cost (tokens, money, ticks of
  latency). Simulation uses them; nothing else does. None of the surveyed simulators has this.
- An LLM call is an oracle. Its outcome set is the classes its output is mapped to (for example
  `tool_call | answer | refusal | error`), not its text.

## 5. Trainable nodes: modelspec lowering and lifting

### 5.1 What modelspec offers today (0.28.0)

- A pure, statically typed language: one `model` (params, `fn`s) and one `train` block per
  `.mspec` file; "declarative, statically typed and pure, does no I/O"
  (`modelspec/website/docs/concepts/language.md:12,17,40-43,92-103`). 22 operators
  (`modelspec/website/docs/reference/operators.md:10`).
- **Lowering exists**: `.mspec` -> AST -> typed HIR -> Tensor IR -> Training IR -> Kernel IR +
  execution DAG -> SPIR-V -> Vulkan (checked, `modelspec/DESIGN.md:160-172`;
  `modelspec/crates/modelspec-core/src/{lower.rs,ir/tensor.rs,ir/training.rs,ir/kernel.rs,autodiff.rs,schedule.rs}`).
- **Lifting does not exist**: no import from another graph format and nothing that turns a
  trained model back into a spec (searched by the reading agent: lift, onnx, safetensors).
- A typed function signature: `fn name(args: Tensor<..>) -> Tensor<..>`; `infer --function <name>`
  runs any model function over dense `<arg>.f32|.i32` files
  (`modelspec/crates/modelspec-cli/src/infer.rs:26-45`; `crates/modelspec-core/src/compile.rs:186-281`).
- A checkpoint `.msckpt` v1: magic, blake3 checksum, JSON manifest of parameters by canonical path
  with dtype and shape, `schema_hash` (`modelspec/DESIGN.md:432-456`;
  `modelspec/spec/domains/checkpoint.yaml`).
- A pure CPU reference interpreter: `evaluate(&TensorProgram, inputs, params) -> outputs`
  (checked, `modelspec/crates/modelspec-runtime/src/cpu/interpreter.rs:238`), with
  `evaluate_seeded` for dropout. It lives in `modelspec-runtime`, whose Vulkan code is behind the
  `vulkan` feature (`modelspec/crates/modelspec-runtime/src/lib.rs:13-19`).
- Discrete operations: only `row_argmax` (`f32[R,C]` -> `i32[R]`). No `bool` dtype ("later",
  checked, `modelspec/DESIGN.md:108-112`), no comparison or `select`, no runtime `if`/`while`
  (`modelspec/DESIGN.md:153`).
- Plans: `epic:population-training` (draft): 100-200 small models trained at once on one GPU and
  selected by fitness, copied and perturbed per generation
  (`modelspec/.engineering/planning/epic/population-training.md`).

### 5.2 A learned node

A `learned` atom declares:

- its ports: `tensor<…>` types, or enums and records that `encode` / `decode` atoms turn into
  `i32` indices and `f32` features;
- a modelspec model: `.mspec` text carried by the spec (or a digest of one), and the name of the
  model function that is the node's forward;
- its weights: a `.msckpt` digest (or none, untrained);
- its training declaration: which upstream ports are features, which port or oracle answer is the
  target, the loss, the optimizer, steps and seed.

### 5.3 Lowering (Gatter -> modelspec)

- Gatter writes the `.mspec` text of the node's model with a `train` block whose `input` and
  `target` types are the node's port types (pure: spec in, text out).
- Training data is a recording: the host runs scenarios or a simulation, records the node's input
  ports and its target per tick, and writes them as dense `<arg>.f32|.i32` files for
  `modelspec train --data` (the format of `infer.rs:26-45`).
- The host runs `modelspec train … --backend vulkan`. Gatter never touches the GPU (G1).
- Training is per node, supervised by the recorded target. No gradient crosses a gate: Gatter's
  discrete atoms have no derivative, and modelspec has no `bool` or `select` to carry one. End-to-end
  training across a circuit is out of scope for v0.1 (Q5).

### 5.4 Lifting (modelspec -> Gatter)

Gatter defines lifting as: from a `.msckpt` manifest and the `.mspec` function signature, derive
the learned node's contract (port types, the `schema_hash`) and pin its weights by digest.

- A checkpoint whose manifest does not match the node's model is refused, as modelspec refuses it
  (`E-CKPT-003`, per the reading agent of `modelspec/DESIGN.md:552`).
- Lifting a trained node back into *gates* (rule extraction, distillation into a `map` table) is
  a later story: for a node whose inputs are a finite enum product, its trained function can be
  tabulated exhaustively by the CPU reference and stored as a `map` atom.

### 5.5 Evaluating a learned node inside the kernel

- Option A (kernel-side, pure): `gatter-modelspec` depends on `modelspec-runtime` without the
  `vulkan` feature and calls the CPU `evaluate` per tick, with the checkpoint's bytes passed in by
  the host. Deterministic; slow for large models.
- Option B (host-side): the learned node is evaluated as an oracle whose answer the host computes
  (CPU or GPU, via `ExecutionPlan`, `modelspec/crates/modelspec-runtime/src/execution.rs:50-130`).
- v0.1 takes A for scenarios and exhaustive exploration, B for large searches.

### 5.6 Variants and population training

A search over variants whose difference is a learned node's hyperparameters or weights is the same
shape as modelspec's `epic:population-training`: N copies perturbed and trained at once on one GPU,
selected by fitness. Gatter supplies the variants and the fitness (7.3); modelspec trains the
population. The epic's open question "separate processes versus a population axis in the Kernel IR"
is modelspec's.

### 5.7 What Gatter needs from modelspec (modelspec stories, not filed)

- `bool` dtype, comparison and `select`, for nodes that mix masks and gates (`DESIGN.md:108`).
- The `infer` signature declaration, refused by the parser today (`language.md:136-143`).
- A serialised, versioned Tensor IR (today printed as text only; the reading agent found no
  `Serialize` in `modelspec/crates/modelspec-core/src/ir/`), if Gatter is to pin a node by IR digest
  instead of by source text.

## 6. How other repositories consume or feed Gatter

### 6.1 inference: a host for streaming circuits

inference has the only built typed-port streaming graph: `NodeImplementation<P>`
(`inference/crates/inference-pipeline/src/node.rs:100-137`), a typed builder
(`inference-pipeline/src/builder.rs:384-390`), validation at declaration and planning
(`inference-pipeline/src/validation.rs:186-215`, `planning.rs:64-206`). It lacks routers,
conditionals and fan-in (goal G-03, `inference/docs/design/inference.md:64`), and its port types
are six modalities (`inference/crates/inference-core/src/model.rs:84-91`).

- Consumes: a Gatter composite becomes one inference node (role `Policy` or `Transform`); each
  envelope is a tick; the node calls the kernel's pure `step`. inference gets router and
  conditional nodes without a second graph engine.
- Feeds: inference node descriptors become Gatter opaque kinds, so a pipeline can be checked and
  simulated as a circuit.

### 6.2 ESS: specification of services, and of Gatter itself

- Gatter's own nouns (component, port, contract, connection, composite, variant, scenario) get an
  ESS domain before code, the spec-first practice of the beyond10x repositories.
- From ESS: the conformance shape (scenarios, the interpreted reference target,
  `ess/crates/verify/ess-conformance/src/interpret.rs:8-22`), the `external:` notion, the
  breaking-diff classifier (`ess/crates/verify/ess-diff/src/compatibility.rs:1-12`).
- To ESS (inferred use): a guard or decision of an ESS entity may be a Gatter composite.

### 6.3 canon: the gate layer it already is

canon's predicates are And/Or/Not over three-valued claims
(`canon/crates/canon/src/model/predicate.rs:142-170`), with `canon check`, `simulate`, `generate`
and `mutate`. canon has no states, no typed ports and no output contracts
(`canon/crates/canon/src/ir/mod.rs:165-245`;
`canon/docs/design/canon-protocol-calculus-design.md:915-924`).

- Gatter's `tri` type and predicate language are canon's: a combinational composite over `tri`
  lowers to a canon claim; a canon claim lifts to a composite.
- Gatter reuses canon's schema narrowing/widening classifier for compatibility (3.4).
- canon says what is legitimate; Gatter says what a circuit computes.

### 6.4 Simulators over ESS models: first consumers of exploration and search

A simulator over an ESS model, running on ESS's interpreted target, exists in a repository that is
not public. It has weighted random walks that are deterministic for any thread count, mutation
points over setup values and faults, and a search in two modes, `fit` (a fitness over KPIs with
gates as hard constraints, hill climbing with restarts) and `exhaustive` (every combination within
a bound, the product size printed first), evaluated in parallel. It refuses specification variants
for now and plans them through ESS's spec mutator.

- Gatter takes that search shape as section 7.
- Such a simulator consumes Gatter's variant and search code for the specification variants it
  defers. Which moves first is Q2.

### 6.5 modelspec

Section 5.

### 6.6 llm-gateway, metaharness: harness nodes, LLM nodes, the first consumer circuit

The owner, 2026-11-26: a real harness wrapped by metaharness becomes "harness_x" behind one
interface, "unified metaharness ctrl + eventing <----> metaharness(claudecode|codex|... + config
w/ baseURL etc) -> (anthropic/… OR llm-gateway) | then llm-gateway with routes either passthrough
for the subscription stuff or vanilla API when api key … <-- this would be a "harness_node" - there
could also be "just" an llm-node but this would not work with subs in the sense that it itself
mimics and pretends to be claude-code".

Two node kinds follow. Both are oracles (section 4) to the kernel: Gatter never runs them; a host
does.

**`harness_node`**: a real agent harness driven through metaharness.

- Control and events: metaharness starts the harness behind one event stream and one command set
  (approve, deny, inject, halt) (`metaharness/README.md:3-16`). Claude Code and Codex are adapters
  today (`metaharness/crates/metaharness-protocol/src/spec.rs:19-31`).
- Config: the run declares the model endpoint. For Claude Code that is `ANTHROPIC_BASE_URL`, for
  Codex a `model_providers` entry with `base_url` (`metaharness/crates/metaharness-claude/src/launch.rs:1546-1558`;
  `metaharness/crates/metaharness-codex/src/launch.rs:1422-1436`).
- Upstream: the vendor directly, or llm-gateway.
- At llm-gateway: a harness on its own subscription login uses a pass-through route
  (`auth_kind = "client"`): the request and the client's own credential are forwarded unchanged,
  and the gateway adds no credential (`llm-gateway/docs/gateway.md:600-618`). A harness on an API
  key uses an ordinary route with the key held by the gateway or the client
  (`llm-gateway/website/docs/guides/store-an-upstream-key.md`).
- Ports: a task in; events and a result out. Outcome set: metaharness's `CloseReason`
  (`completed`, `budget`, `killed`, `error`, `steer-halt`,
  `metaharness/crates/metaharness-protocol/src/event.rs:944`). KPIs: tokens, cost, turns, wall time
  (section 7.4).
- Gap: metaharness's `--model-endpoint` requires `--credentials none` today
  (`metaharness/crates/metaharness-protocol/src/spec.rs:335-341`), so "own subscription login +
  gateway pass-through" needs a metaharness change.

**`llm_node`**: one direct model call.

- A single turn through llm's `Model::turn` port (`llm/crates/llm-core/src/port.rs:105-123`),
  to a provider or through llm-gateway.
- Credential: an API key, or none for a self-hosted model. **Never a subscription credential**: a
  direct call carrying one would present itself as a vendor's own client. llm has an account kind
  that does this (`subscription-oauth`, `llm/docs/messages.md:122-130`); an `llm_node` contract
  refuses that account kind.
- Ports: a request record (or `schema<…>`) in; an answer out. Outcome set: answer, tool call,
  refusal, error. KPIs: tokens, cost, latency.

The rule between them: subscription use only ever happens inside a `harness_node`, where the
vendor's own client sends its own credential; everything else is an `llm_node` on an API key.

**The first consumer circuit.** The route choice of that path (request fields in; a route enum,
pass-through or translate, and the node kind out) is the first composite to write. The gateway
crate stays dependency-free (`llm-gateway/docs/gateway.md:11-19`); a policy evaluated by Gatter
would enter through the gateway's narrow ports (`CallerReview`,
`llm-gateway/crates/llm-gateway/src/review.rs:118`) from its binary, never as a dependency of the
gateway crate (inferred; not agreed with that repository).

### 6.7 Others

- swarm: its declared `Port{name, schema_id, required}` and `Connection{…, delivery, on_failure}`
  (`swarm/src/core/domains/blackbox.yaml:81-89,311-350`) are the closest vocabulary; Gatter takes
  the names where the meaning is the same.
- ratchet: `better: lower|higher` measurements and gates (`ratchet/docs/design.md:28,144,192,218`)
  are the search's fitness and hard constraints.
- runbook experiments: the host-side pattern for variants whose oracles are real LLM calls
  (`runbook/docs/workflow-metrics.md:192,236-237`).

## 7. Behaviour spec: simulation and sparse mutated clones

### 7.1 Exploration

- **Exhaustive** for finite circuits (every port type bounded, every oracle with a finite outcome
  set): breadth-first over (state, input, oracle answer) up to a tick bound; reports reachable and
  unreachable states and outputs, and property failures with the shortest trace. The role of
  `canon check` / `canon simulate` (`canon/crates/canon/src/check/mod.rs:1-60`,
  `canon/crates/canon/src/simulate/mod.rs:1-125`).
- **Seeded random walks** for large or unbounded circuits: weighted toward unreached outputs,
  deterministic for any thread count, shrinking on failure, as ESS's generated explorer does
  (`ess/crates/verify/ess-conformance/src/go/explore.go:1-30`).
- **Properties**: invariants each tick, reachability targets, bounded response ("within n ticks of
  X, Y"), state patterns over rows.
- Oracles are enumerated in exhaustive mode and sampled from their distribution in random mode.

### 7.2 Variants: sparse structural clones

- A **mutation point** is a named place that may vary: an atom's kind (among kinds with a
  compatible contract), a parameter, a connection (rewired only where 3.4 admits it), a guard (an
  `fsm` transition's predicate present or absent, a threshold), a behaviour-tree child order, a
  learned node's hyperparameters or weights.
- A **variant** is the base circuit plus a sparse delta: a list of edits at mutation points. It
  stores only the delta; every unchanged composite is shared by digest.
- **Shared evaluation**: evaluation is pure, so a sub-composite with the same digest, state and
  inputs in two variants is computed once per tick and reused. N variants that differ in one leaf
  cost one shared circuit plus N leaves.
- A variant that fails the check (type, contract, combinational loop) is recorded *stillborn*, as
  ESS records uncompilable spec mutants (`ess/crates/verify/ess-conformance/src/mutate.rs:3163-3188`).

### 7.3 Search

- `exhaustive`: every combination of mutation-point values within a bound; the product size is
  printed first and refused over a limit.
- `fit`: a fitness over measured KPIs with gates as hard constraints; hill climbing with restarts or
  a small evolutionary loop picks the next variants from the best so far.
- Every visited variant is written with its KPIs and its delta: a result is a table and a front.
- Parallelism: the kernel evaluates a batch of variants as a pure function of the batch; the host
  chooses threads, and GPUs for learned nodes. The kernel spawns nothing (G1).
- The same mutation operators score a scenario set (killed / survived): one engine serves "find a
  better circuit" and "is this test suite strong".

### 7.4 Quality composes

The owner, 2026-11-26: "individual lower components start having kpis the more high level they
become. and a composition of the units is also a composition of its quality? there will be an
evaluatable quality for the composed system, genetic algorithms, variant generation".

- **KPIs grow with level.** An atom has structural figures only: gate count, depth, state count,
  latency in ticks. A composite adds the KPIs of its role: a router has accuracy against labelled
  scenarios; an `llm_node` has tokens, cost and latency; a `harness_node` has turns, cost and wall
  time; a behaviour tree has success rate and ticks to success.
- **A KPI is declared** in a component's contract: a name, a unit, `better: lower | higher` (the
  form ratchet uses, `ratchet/docs/design.md:28,144`), optional bounds as guarantees ("p95 latency
  ≤ 5 ticks"), and how it composes.
- **How a KPI composes** is declared per KPI kind, as in reliability block diagrams:
  - additive in series, max in parallel: latency, ticks;
  - additive everywhere: cost, tokens, gate count;
  - multiplicative in series, `1 − Π(1 − r)` over a `fallback`: success probability, given
    independent failures;
  - expectation over a `mux`: any of the above, weighted by the branch distribution of the
    selecting input (from an oracle's declared distribution, or measured);
  - `measured`: no rule (accuracy, emergent behaviour of a feedback loop). The value comes only
    from simulation or scenarios.
- **Derived and measured are both kept.** A composite's KPI is derived from its children by the
  rule, and measured by simulation (7.1). A gap between the two beyond a declared tolerance is a
  finding: a hidden dependency between children, or a wrong independence assumption.
- **The composed quality is the fitness.** A system's quality is a vector of its top-level KPIs;
  the search (7.3) reads it with weights, and the gates (hard bounds) refuse a variant outright.
  Every visited variant keeps the vector, so the result is a Pareto front, not one number.
- **Genetic algorithms over sparse deltas.** A variant's genome is its delta (7.2): values at named
  mutation points. Crossover takes the edits of two parents at disjoint mutation points; mutation
  changes one point's value; a child that fails the check is stillborn. Because deltas are sparse
  and sub-circuits shared by digest, a population of N variants costs the shared circuit plus the
  N deltas (7.2).
- **Learned nodes in the population** train through modelspec's population plan (5.6); their
  KPIs (loss, accuracy, inference cost) enter the vector like any other.
- **Cost of a KPI.** A derived KPI costs nothing to evaluate; a measured KPI costs a simulation, and
  a KPI of a `harness_node` or `llm_node` costs real model calls when not sampled from a declared
  distribution. The search prefers derived KPIs to prune, and measures only the front.

## 8. Shape of the code (planned)

- `crates/gatter-core`: types, contracts, atoms, composites, check, `step`. IO-free, enforced by a
  test in the style of `llm-gateway/crates/llm-gateway/tests/dependency_boundary.rs` (declared and
  transitive dependencies against a forbidden list, with positive controls) plus a source scan for
  `std::fs`, `std::net`, `std::time`, `std::env`, `std::process`, `std::thread`.
- `crates/gatter-explore`: exhaustive and random exploration, properties. IO-free.
- `crates/gatter-variant`: mutation points, deltas, shared evaluation, search. IO-free; parallel
  only through a host-provided executor trait.
- `crates/gatter-modelspec`: lowering to `.mspec` text, lifting from a checkpoint manifest, CPU
  evaluation (5.5 A). IO-free: bytes in, values out.
- `crates/gatter-cli`: the one crate with IO: reads files, runs scenarios, runs a search on
  threads, runs `modelspec train` as a subprocess.
- `spec/`: the ESS domain of Gatter's nouns; `spec/meta/` the meta-spec; `spec/atoms/` the atoms
  with their scenarios.

Rust, as every kernel of the beyond10x repositories. No Python in anything committed.

## 9. Open questions

Decided 2026-11-26 by the owner: the name `gatter`, a public repository.

- Q1 Document format: YAML like ESS and canon, or a small text language like `.mspec`.
- Q2 Which moves first: an existing ESS simulator consuming Gatter's search, or Gatter taking
  over that simulator's search code (6.4).
- Q3 Whether ESS entity guards may be Gatter composites (6.2), and whether canon claims and Gatter
  combinational composites share one predicate crate (6.3).
- Q4 How far contract implication is decided (3.4, rule 2) before it becomes an obligation for
  conformance.
- Q5 End-to-end training across gates (soft gates, straight-through estimators) or per-node
  training only (5.3).

## 10. First stories (proposed, not filed)

1. ESS domain for Gatter's nouns; meta-spec of the atom kinds; scenarios for `and`, `or`, `not`,
   `register`, `fsm`.
2. `gatter-core`: types, compatibility rule 1, combinational-loop check, `step`, scenario runner;
   the IO-free boundary test.
3. Behaviour-tree atoms compiled to `fsm`; one composite with scenarios.
4. Exhaustive exploration of a finite circuit; one property.
5. Variants: mutation points, sparse deltas, `exhaustive` search, stillborn record.
6. A `learned` node lowered to `.mspec`, trained by `modelspec train` from the CLI, lifted from its
   checkpoint; one scenario over the trained node.
7. The LLM route-selection composite (6.6) as the first consumer circuit, with `harness_node` and
   `llm_node` as opaque kinds.
8. KPI declarations and composition rules (7.4); derived and measured values side by side on one
   composite; a genetic search over sparse deltas.
