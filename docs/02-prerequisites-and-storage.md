## Prerequisites and persistent storage

The following steps assume a fresh, single-node Linux Kubernetes environment. We will use the namespace `kafka` throughout the tutorial.

Before starting, you need:

- A working Kubernetes cluster with a schedulable node and functioning cluster DNS.
- `kubectl` access with permission to create namespaces, PersistentVolumes, and workloads.
- Access to the node’s filesystem to prepare Kafka’s data directory.
- Enough free disk space for the example’s 50 GiB storage allocation, plus operating-system requirements.
- Access to the container images `apache/kafka:4.3.1` and `provectuslabs/kafka-ui:v0.7.2`.

The Kafka image version follows the deployment used for this guide and is listed in Apache’s [Docker documentation](https://kafka.apache.org/43/getting-started/docker/).

### 1. Check the cluster and create the namespace

~~~bash
kubectl get nodes -o wide
kubectl get nodes -L kubernetes.io/hostname
kubectl create namespace kafka
~~~

Confirm that the intended node is `Ready` and can accept workloads. Record its `kubernetes.io/hostname` label; we will use that value to associate storage with the correct node. If the namespace already exists, skip its creation.

### 2. Prepare the host directory

Kafka will store data in `/var/lib/kafka/data` inside its container. The volume will map this to `/mnt/data/kafka` on the node.

First, inspect the user and group used by the Kafka image:

~~~bash
kubectl run kafka-image-check \
  --namespace kafka \
  --image=apache/kafka:4.3.1 \
  --restart=Never --rm -i \
  --command -- id
~~~

On the selected Kubernetes node, create the directory with ownership matching that output. For a reported UID and GID of `1000:1000`, use:

~~~bash
sudo install -d -m 0770 -o 1000 -g 1000 /mnt/data/kafka
df -h /mnt/data/kafka
~~~

If the image reports different IDs, substitute those values.

This example uses `hostPath` storage, which is tied to the node’s filesystem. Declaring `50Gi` in the PV does not create a dedicated disk or enforce a 50 GiB directory quota. Actual disk space must be monitored. [Kubernetes volume documentation](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)

### 3. Define the PersistentVolume

Save the following as `kafka-pv.yaml`. Replace `REPLACE_WITH_NODE_HOSTNAME` with the hostname label recorded earlier.

~~~yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-kafka-0
spec:
  capacity:
    storage: 50Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/kafka
    type: Directory
  claimRef:
    namespace: kafka
    name: kafka-data-kafka-0
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - REPLACE_WITH_NODE_HOSTNAME
~~~

The `claimRef` reserves this volume for the intended claim. The node affinity constrains workloads using it to the node holding the data. `Retain` preserves the volume and its data when the claim is deleted; it does not provide a backup or automatic failover. [PersistentVolume documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

### 4. Define the PersistentVolumeClaim

Save this as `kafka-pvc.yaml`:

~~~yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-data-kafka-0
  namespace: kafka
  labels:
    app: kafka
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  storageClassName: ""
  resources:
    requests:
      storage: 50Gi
  volumeName: pv-kafka-0
~~~

The explicit `storageClassName: ""` requests storage without a StorageClass, avoiding selection of a default class. `volumeName` identifies the intended PV. [StorageClass matching](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#class-1)

The claim name follows the StatefulSet naming pattern: claim template `kafka-data`, StatefulSet `kafka`, and pod ordinal `0`. In the later StatefulSet example, we will use a matching claim template requesting 50 GiB.

### 5. Apply and verify the storage

~~~bash
kubectl apply -f kafka-pv.yaml
kubectl apply -f kafka-pvc.yaml

kubectl get pv pv-kafka-0
kubectl get pvc kafka-data-kafka-0 -n kafka
~~~

Before moving on, verify that both resources report `Bound` and that the PVC references `pv-kafka-0`. If the claim remains `Pending`, inspect its events:

~~~bash
kubectl describe pvc kafka-data-kafka-0 -n kafka
~~~

These commands establish the storage binding. Kafka’s ability to write to the mounted directory will be checked after the broker starts.

