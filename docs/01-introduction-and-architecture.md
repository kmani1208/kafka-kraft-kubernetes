# Setting Up Apache Kafka in KRaft Mode on Kubernetes: A Practical Guide

Getting a Kafka pod running is one part of the setup. Connecting applications, configuring advertised listeners, preserving data, and verifying that producers and consumers can communicate are equally important.

While setting up Kafka in KRaft mode on Kubernetes, I encountered situations where Kafka UI displayed messages but a consumer remained waiting. Troubleshooting these situations helped me understand how listener configuration, Kubernetes Services, and client connectivity fit together.

This guide brings those configuration details and practical lessons into one walkthrough. It is intended for developers, DevOps engineers, data engineers, and anyone interested in deploying Kafka with a basic understanding of Kubernetes.

## What we will build

We will deploy a single Kafka broker using KRaft, with persistent storage, access from inside and outside Kubernetes, and Kafka UI for inspecting topics and messages.

KRaft allows Kafka to manage its metadata without ZooKeeper. Kafka processes can serve as brokers, controllers, or both. In this setup, one process performs both roles:

- The **broker** handles client requests and stores topic records.
- The **controller** manages cluster metadata and coordinates activities such as partition leader elections.

Combined roles make a small development environment simpler to operate. This guide uses a **static controller quorum**, with one controller explicitly configured as a voter. [Apache Kafka KRaft documentation](https://kafka.apache.org/43/operations/kraft/)

## How the Kubernetes components fit together

| Component | Purpose |
|---|---|
| **Kafka StatefulSet** | Runs Kafka with a stable pod identity and a persistent storage claim. |
| **Headless Service** | Provides the DNS identity used in the broker and controller configuration. |
| **Kafka Service** | Provides an internal connection endpoint and exposes the external listener through NodePort. |
| **PersistentVolume and PersistentVolumeClaim** | Connect Kafka’s data directory to storage on the Kubernetes node. |
| **Kafka UI Deployment and Service** | Provide a browser interface for inspecting the cluster, topics, and messages. |

The StatefulSet, headless Service, and storage claim work together to preserve the pod’s identity and storage association across replacement. The underlying data in this example remains on the same node. [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

## Understanding the three listeners

The configuration separates communication into three named listeners:

| Listener | Broker port | Purpose |
|---|---:|---|
| **INTERNAL** | `9092` | Connections from applications inside Kubernetes, including Kafka UI. |
| **CONTROLLER** | `9093` | KRaft controller communication. |
| **EXTERNAL** | `9094` | Client connections arriving through NodePort `30092`. |

An external application connects to `<NODE_IP>:30092`, and the Service forwards that connection to Kafka’s EXTERNAL listener on port `9094`.

Kafka also returns broker addresses to clients through its metadata. Those **advertised addresses must be reachable from the client’s network**. We will examine this distinction when configuring internal and external access. [Kafka broker configuration](https://kafka.apache.org/43/configuration/broker-configs/#advertised.listeners)

This walkthrough is a development and learning setup. One broker and one controller provide no redundancy, and the configured PLAINTEXT connections do not encrypt traffic. These boundaries matter when adapting the example for another environment. [Kafka listener documentation](https://kafka.apache.org/43/security/listener-configuration/)
