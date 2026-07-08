# Caelius — System Design

## Overview

Caelius is a distributed spiking neural network simulator. Individual neurons
are modeled as concurrent processes that communicate via spike events, forming
a biologically-inspired network whose emergent behavior is captured, streamed,
and analyzed by a machine learning layer.

The project sits at the intersection of computational neuroscience, distributed
systems engineering, and machine learning. It is not a deep learning framework.
It is a simulation of how biological neurons actually behave, and an
infrastructure for studying what that behavior produces at scale.

---

## Architecture

Caelius is composed of three distinct layers, each with a clearly defined
responsibility:

### 1. Neuron Layer (Go)
Models individual neurons as concurrent goroutines using the Leaky
Integrate-and-Fire (LIF) model. Each neuron:
- Maintains a membrane potential that decays toward resting state over time
- Receives input signals from connected neurons via channels
- Fires an action potential (spike) when membrane potential crosses a threshold
- Enters a refractory period after firing during which it cannot fire again
- Broadcasts its spike to downstream neurons and to the Kafka event stream

Neurons are connected at startup via a configurable topology. The simulation
is event-driven: neurons only compute when they receive input or fire. There
is no global clock or tick loop.

### 2. Transport Layer (Kafka)
All spike events are published to a Kafka topic by the Go simulation layer.
Kafka was chosen over ZeroMQ and NATS for the following reasons:
- Spike events are persistent and replayable, enabling post-hoc analysis
  without re-running the simulation
- The same spike train data can be analyzed by multiple consumers
  simultaneously or at different points in time
- Real biological spike train datasets can be injected into the same pipeline
  for comparison against simulated data
- Kafka's presence in production distributed systems makes it a meaningful
  engineering credential

Each spike event is a Kafka message containing: neuron ID, simulation
timestamp, and membrane potential at time of firing.

### 3. Analysis Layer (Python)
Consumes the Kafka spike stream and performs:
- Real-time ingestion and storage of spike train data as time series
- Visualization via raster plots (neuron ID vs. simulation time)
- Machine learning analysis of emergent firing patterns

---

## Key Design Decisions

### Event-Driven vs. Tick-Based Simulation
Caelius uses an event-driven simulation model. Neurons only compute when
they receive a spike or fire one. This decision was made because:
- It is more biologically accurate: real neurons are idle until stimulated
- It maps naturally to Go's goroutine and channel model
- It makes every Kafka message meaningful: a message means something happened
- It produces a harder and more interesting distributed systems problem
  around timing consistency

The tradeoff accepted: timing consistency across goroutines requires explicit
simulation timestamps rather than wall clock time.

### Neuron Model: Leaky Integrate-and-Fire
The LIF model was chosen as the neuron abstraction because:
- It is mathematically grounded and well studied in computational neuroscience
- It is simple enough to implement quickly without sacrificing biological
  plausibility
- It produces realistic spike trains that are meaningful for ML analysis
- It is a natural stepping stone toward more complex models (Hodgkin-Huxley)
  if the project evolves in that direction

### Language Choice: Go for Neurons
Go was chosen for the neuron simulation layer because:
- Goroutines and channels are a natural fit for concurrent event-driven neurons
- Go's performance characteristics suit simulation workloads
- It represents a deliberate investment in production Go experience

### Language Choice: Python for Analysis
Python was chosen for the analysis layer because:
- It is the author's strongest language
- The scientific Python ecosystem (NumPy, scikit-learn, matplotlib) is
  the standard for this class of analysis work

### Input Signal: Independent Poisson Spike Trains (MVP)
External input is generated as a homogeneous Poisson process — spike
intervals drawn from the exponential distribution, producing exact
continuous-time spike times rather than binning time into discrete steps.
This was chosen over a fixed periodic pulse because it's the formulation
that's actually compatible with the event-driven, non-tick-based simulation
model: a discrete-time binned approximation would quietly reintroduce a
tick loop.

4 of the 20 neurons receive external input (each an independently-sampled
Poisson process, same rate), fanning out to the remaining 16. Independent
draws were chosen over a single shared spike train specifically to avoid
baking artificial synchrony into the input — since a future ML layer may
be asked to detect emergent synchrony, the input itself must not
manufacture it.

Deferred: correlated/shared input, and a time-varying (inhomogeneous)
rate function. See Phased Complexity Roadmap below.

### Network Topology: Random, Feed-Forward DAG (MVP)
Topology is random rather than structured for MVP — the goal is a working
network that propagates spikes correctly, not a specific graph shape.
Connections are strictly feed-forward (a random DAG, generated by assigning
each neuron a random topological rank and only permitting edges from lower
to higher rank) — no cycles for MVP. The 4 external-input neurons are the
rank-0 source nodes. Partial reachability (some of the 16 downstream
neurons not receiving a path from an input neuron in a given random
topology) is acceptable for MVP.

Deferred: recurrent connections (cycles). See Phased Complexity Roadmap
below.

---

## Phased Complexity Roadmap

A pattern emerged across several independent design decisions: MVP
consistently takes the acyclic, non-adaptive, uncorrelated version of a
given axis, with each later phase adding one axis of biological realism.
Recording it explicitly so future open questions can be answered against
the same principle instead of re-derived from scratch:

| Axis | MVP | Phase 2 | Phase 3 |
|---|---|---|---|
| Neuron model | LIF | LIF | Hodgkin-Huxley (possible) |
| Input signal | Independent Poisson, homogeneous rate | — | Correlated input, inhomogeneous rate |
| Topology | Random, feed-forward (DAG) | — | Recurrent (cycles allowed) |
| Synaptic weights | Fixed at startup | — | Plastic — spike-timing-dependent plasticity (STDP) |
| Analysis | Raster plot visualization | ML on spike trains | ML on pathway formation under STDP |

The driving research question behind phase 3 is whether spiking neural
networks develop preferred signal pathways through repeated use, the way
biological neural pathways strengthen with use (Hebbian learning / STDP /
long-term potentiation). This requires synaptic plasticity, which MVP does
not have — so MVP-era topology and input decisions are intentionally not
optimized around pathway-tracking; that consideration only becomes live
again once STDP is being designed.

---

## Open Questions

- What does the ML analysis layer classify or detect? Synchronized vs.
  unsynchronized firing? Healthy vs. degraded network state? (Deferred to
  phase 2 — see Phased Complexity Roadmap. Likely candidate given the
  phase 3 research question: detecting whether preferred pathways have
  formed under STDP.)
- What does the STDP / plasticity rule look like for phase 3, and how is
  "a preferred pathway formed" operationalized as something measurable?

---

## MVP Scope

- 20 neurons connected in a random, feed-forward (DAG) topology
- Independent homogeneous Poisson-process input signal injected into 4 of
  the 20 neurons
- Spike events streamed to Kafka
- Python consumer produces a real-time raster plot
- No ML layer in MVP — visualization only, ML is phase two
- No synaptic plasticity in MVP — fixed weights, STDP is phase three

---

## Stack

| Layer | Technology |
|---|---|
| Neuron simulation | Go |
| Message transport | Apache Kafka |
| Analysis and ML | Python |
| Visualization | Matplotlib / Streamlit (TBD) |
| Containerization | Docker Compose |