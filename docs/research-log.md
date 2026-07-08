# Caelius Research Log

---

## July 6, 2026

### Project Inception

Caelius was conceived tonight as a distributed spiking neural network
simulator. The name comes from the Latin word for "of the sky or heavens,"
chosen to reflect the dual inspiration behind the project: the cosmos of
space (LEO satellite systems) and the cosmos of the brain (neural dynamics).

### Motivation

The project emerged from a desire to build something at the intersection of
genuine scientific interest and distributed systems engineering. Existing
personal projects (Hephaestus, CLI-Copilot) demonstrate backend engineering
competence. Caelius is intended to demonstrate something harder to fake:
curiosity applied to a problem that doesn't have an obvious right answer.

### Architecture Decisions Made Tonight

The following foundational decisions were made and documented in DESIGN.md:

- **Simulation model:** Event-driven, not tick-based. Neurons only compute
  when they receive input or fire. More biologically accurate, more
  interesting as a distributed systems problem.
- **Neuron model:** Leaky Integrate-and-Fire (LIF). Mathematically grounded,
  biologically plausible, implementable without sacrificing the interesting
  parts.
- **Neuron layer:** Go. Goroutines and channels are a natural fit for
  concurrent event-driven neurons.
- **Transport layer:** Kafka. Chosen over ZeroMQ and NATS for persistence
  and replayability. Every spike is recorded. Simulations can be replayed
  and analyzed differently after the fact.
- **Analysis layer:** Python. Scientific ecosystem, strongest language.

### Open Questions Going Into First Build Session

1. What is the input signal? What external stimulus drives network activity?
2. What does the initial network topology look like?
3. What does the ML layer classify or detect?

### Next Session Goals

- Define the Kafka message schema for a spike event
- Write the first Go struct representing a neuron
- Get Kafka running locally via Docker Compose

---

## July 7, 2026

### Resolved: Input Signal and Topology Open Questions

Worked through open questions 1 and 2 from the previous session.

**Input signal:** Considered fixed periodic pulses vs. Poisson-process spike
generation. Researched how Poisson spike generators work — the key
distinction turned out to be continuous-time exact-event sampling (spike
intervals drawn from the exponential distribution) vs. discrete-time binned
approximation (checking a probability per time-bucket, which only converges
to a true Poisson process as bucket width goes to zero). The continuous
formulation is the one actually compatible with the event-driven,
non-tick-based simulation model — a binned approach would quietly
reintroduce a tick loop. Decided: homogeneous Poisson process, independent
draws (not a shared spike train) into 4 of the 20 neurons. Independent
draws were chosen after researching correlated vs. independent input in
spiking neural networks — correlated input drives the precise pre/post
spike timing needed for STDP-driven long-term potentiation, while
independent input is the standard choice for modeling background noise
and testing network robustness absent a learning rule. Since MVP has no
plasticity mechanism, independent input is the appropriate MVP choice;
correlated input is deferred to phase 3, paired with STDP.

**Topology:** Locked in random (not structured) for MVP, strictly
feed-forward (no cycles), partial reachability acceptable. Noted for
implementation: generating a random graph that's also guaranteed acyclic
requires rank-ordered generation (assign each neuron a random topological
rank, only allow edges from lower to higher rank), not generate-then-discard
cycle checking. The 4 input neurons become the natural rank-0 source nodes.

### A Real Research Question Surfaced

Underneath the input-neuron-count discussion, a more specific research
question emerged than what was written in the original open question 3:
**do spiking neural networks develop preferred signal pathways through
repeated use, the way biological neural pathways strengthen with use?**
This is a Hebbian learning / spike-timing-dependent plasticity (STDP)
question, and current MVP scope has no mechanism for synaptic weights to
change at all — connections are fixed at startup. This can't be
investigated until plasticity exists in the simulator.

### Decision: Phased Complexity Roadmap

Formalized a pattern that had already been showing up implicitly across
several decisions (LIF before Hodgkin-Huxley, etc.): MVP takes the acyclic,
non-adaptive, uncorrelated version of every axis of the design, and each
subsequent phase adds one axis of biological realism. Recorded as a table
in DESIGN.md. Concretely: STDP/synaptic plasticity and correlated input are
now explicitly phase 3, paired together, since correlated input only does
anything useful once there's a learning rule for it to drive.

Open question 3 (what the ML layer classifies) is deliberately left open
and deferred to phase 2 — no need to answer it to unblock MVP work, and the
phase 3 research question (pathway detection) will likely shape it once
STDP is actually being designed.

### Next Session Goals

- Define the Kafka message schema for a spike event
- Write the first Go struct representing a neuron
- Get Kafka running locally via Docker Compose