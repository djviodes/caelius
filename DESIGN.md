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

---

## Open Questions

- What is the input signal? What external stimulus triggers the first spike
  and drives ongoing network activity?
- What does the initial network topology look like? Random? Structured?
  How many neurons in the MVP?
- What does the ML analysis layer classify or detect? Synchronized vs.
  unsynchronized firing? Healthy vs. degraded network state?

---

## MVP Scope

- 20 neurons connected in a configurable random topology
- External periodic input signal injected into a subset of neurons
- Spike events streamed to Kafka
- Python consumer produces a real-time raster plot
- No ML layer in MVP — visualization only, ML is phase two

---

## Stack

| Layer | Technology |
|---|---|
| Neuron simulation | Go |
| Message transport | Apache Kafka |
| Analysis and ML | Python |
| Visualization | Matplotlib / Streamlit (TBD) |
| Containerization | Docker Compose |