# AGENTS.md: gatter

Gatter is an IO-free specification kernel: components with typed contracts, composed from logic
gates up to state machines and behaviour trees, with oracles for dynamic parts, learned nodes
trained through modelspec, conformance scenarios, state-space exploration and a search over sparse
structural variants. Humans start at [`README.md`](README.md); the design is
[`docs/design/gatter-v0.1.md`](docs/design/gatter-v0.1.md).

## State

Design only (2026-11-26). No code, no ESS specification, no planning store. Public repository.

## Map

| path | what |
|---|---|
| `docs/design/gatter-v0.1.md` | the design: model, oracles, learned nodes, consumers, search |
| `Taskfile.yml` | `task check`: the checks that exist for a design-only repository |

## Owns

- The component model: types, contracts, compatibility, atoms, composites, the meta-spec.
- Synchronous tick evaluation of a circuit, scenarios and their runner.
- Exploration (exhaustive, seeded random walks), properties, variants, search.
- Lowering of learned nodes to `.mspec` text and lifting from a modelspec checkpoint.

## Does not own

- IO of any kind in the kernel crates: files, sockets, clocks, environment, processes, threads,
  GPUs. Hosts own it (design § 2, G1). Only `gatter-cli` does IO.
- Service specification: ESS. Legitimacy, evidence and authority: canon. Streaming runtime:
  inference. Model language, compiler, training and GPU: modelspec.

## Rules

- **IO-free kernel.** A kernel crate that gains a dependency or a `std` module that can do IO is a
  defect. The boundary test (design § 8) holds it once code exists.
- **Spec first.** Gatter's nouns get an ESS domain in `spec/` before code (design § 10, story 1).
- **No Python** in anything committed.
- Builds and tests go through [`slot`](https://github.com/beyond10x/slot). The test step ends
  within 2 minutes and no test runs over 60 s. Exploration and search tests use small bounded
  circuits; a full search runs in a `slow` lane, never in the test step.
- **Public repository.** No person name, no non-public repository or service name, no local path
  and no reference to private instruction files in any file or commit message.
