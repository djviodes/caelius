# Caelius

> The cosmos of the brain.

Caelius is a distributed spiking neural network simulator built at the
intersection of computational neuroscience, distributed systems, and machine
learning.

Individual neurons are modeled as concurrent Go goroutines using the Leaky
Integrate-and-Fire model. When a neuron fires, its spike is published to a
Kafka event stream. A Python analysis layer consumes that stream in real time,
visualizing emergent network behavior and running machine learning models
against the spike train data.

This is not a deep learning framework. It is a simulation of how biological
neurons actually behave, and an infrastructure for studying what that behavior
produces at scale.

## Stack

| Layer | Technology |
|---|---|
| Neuron simulation | Go |
| Message transport | Apache Kafka |
| Analysis and ML | Python |
| Containerization | Docker Compose |

## Architecture
[Go Neurons] → [Kafka] → [Python Analysis]

Each neuron is a goroutine. Each spike is a Kafka message. Each message
is a data point in an emergent system that no single component fully controls.

## Status

Early development. Architecture established, implementation in progress.

## Author

David Viodes