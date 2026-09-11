# Kafka KRaft on Kubernetes

A practical guide to a single-broker Apache Kafka KRaft deployment on Kubernetes, developed section by section.

**Status: work in progress.** This initial draft contains the introduction, architecture explanation, prerequisites, and persistent storage examples. It is not yet a complete Kafka deployment package.

## Read the completed sections

1. [Introduction and architecture](docs/01-introduction-and-architecture.md)
2. [Prerequisites and persistent storage](docs/02-prerequisites-and-storage.md)

## Available manifests

| File | Purpose |
|---|---|
| [kafka-pv.yaml](manifests/kafka-pv.yaml) | A 50 GiB hostPath PersistentVolume reserved for the Kafka claim, with node affinity. |
| [kafka-pvc.yaml](manifests/kafka-pvc.yaml) | The matching 50 GiB claim in namespace `kafka`. |

Follow the prerequisites and directory-preparation steps in section 2 first. Replace `REPLACE_WITH_NODE_HOSTNAME` in the PV with the actual node hostname label.

The storage section's apply commands assume the YAML files are in your working directory. In this repository, change into `manifests/` before running those commands.

## Validation so far

- The two YAML manifests parse successfully.
- PV/PVC names, namespace, capacity, and StorageClass settings agree.
- The manifests match the code blocks in the storage section.
- No deployment or runtime test has been performed while preparing this draft.

## Still to be written

- Kafka Services, listeners, and StatefulSet.
- Kafka UI deployment and access.
- Producer/consumer verification.
- Troubleshooting, with evidence-backed outcomes.
- Final article assembly and LinkedIn summary.

## Scope

The examples target a fresh single-node Linux Kubernetes lab, using the neutral namespace `kafka`. Storage is local to the selected node. This is a learning/development configuration with no broker/controller redundancy.

The eventual publication sequence is GitHub, DEV Community, then LinkedIn.

## Issue workflow

The remaining work is tracked in these GitHub issues:

- [#1: Document Kafka Services, listeners, and StatefulSet](https://github.com/kmani1208/kafka-kraft-kubernetes/issues/1)
- [#2: Document Kafka UI deployment and access](https://github.com/kmani1208/kafka-kraft-kubernetes/issues/2)
- [#3: Add producer, consumer, and persistence verification](https://github.com/kmani1208/kafka-kraft-kubernetes/issues/3)
- [#4: Write troubleshooting from the recorded deployment experience](https://github.com/kmani1208/kafka-kraft-kubernetes/issues/4)
- [#5: Assemble and review the complete DEV Community article](https://github.com/kmani1208/kafka-kraft-kubernetes/issues/5)
- [#6: Publish the reviewed article on DEV Community](https://github.com/kmani1208/kafka-kraft-kubernetes/issues/6)
- [#7: Prepare and publish the LinkedIn summary](https://github.com/kmani1208/kafka-kraft-kubernetes/issues/7)

Complete one section at a time and pause for review. Reference its issue in the commit message. Close the issue as completed when its acceptance criteria are met, recording the relevant commit and validation evidence or publication URL. Keep unverified runtime checks clearly identified.
