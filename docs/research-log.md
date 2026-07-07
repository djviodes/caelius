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

- Answer the input signal question
- Define the Kafka message schema for a spike event
- Write the first Go struct representing a neuron
- Get Kafka running locally via Docker Compose