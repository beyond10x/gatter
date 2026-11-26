# Gatter

Gatter describes behaviour as a circuit: small components with typed inputs and outputs, wired
together when their contracts fit, from logic gates up to state machines and behaviour trees.
Some parts are deterministic; some are dynamic (an LLM call, a person) and are declared by their
possible outcomes; some are small models trained on a GPU through
[modelspec](https://github.com/beyond10x/modelspec). Gatter checks a circuit, runs it tick by tick
against scenarios, explores what it can reach, and searches over slightly changed copies of it.

The kernel does no IO. It takes text and values and returns text and values; a host program
supplies time, randomness, files, threads and GPUs.

**Status:** design only. Nothing is built. Read
[`docs/design/gatter-v0.1.md`](docs/design/gatter-v0.1.md).

The name is German for logic gate.
