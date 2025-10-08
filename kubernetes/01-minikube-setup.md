## Setup Kubernetes cluster

### Kubernetes Hierarchy

#### 1. Cluster

- A Kubernetes Cluster = the entire system.
- It consists of:
  - Control Plane (brain that manages the cluster).
  - Nodes (machines that run workloads).
- It exposes a single API endpoint (kubectl) to manage everything.

**Think of the Cluster as the city.**

#### 2. Nodes

- A Node = one machine (VM, physical server, or container in Minikube).
- Each node has:
  - kubelet → agent that talks to the control plane.
  - kube-proxy → handles networking.
  - Container runtime (like Docker, containerd, CRI-O).


**Think of a Node as a building inside the city.**

#### 3. Pods

- A Pod = the smallest deployable unit in Kubernetes.
- A Pod can contain one or more containers that run together (usually 1 container per Pod).
- Pods live on Nodes.
- Pods are ephemeral → they can die anytime, and Kubernetes restarts them.

**Think of a Pod as a room inside the building where actual work happens.**

```txt
Cluster
│
├── Node A (application + control plane)
│     ├── Pod 1 (student-api container)
│     └── Pod 2 (frontend container)
│
├── Node B (database)
│     └── Pod 3 (MySQL container)
│
└── Node C (dependent services)
      ├── Pod 4 (Prometheus container)
      └── Pod 5 (Vault container)
```
---

### 1. Cluster, Nodes, and Pods

- Cluster → Entire Kubernetes system (control plane + worker nodes).
- Node → A machine (VM/physical) that runs workloads.
- Control Plane Node → manages the cluster.
- Worker Node → runs workloads (Pods).
- Pod → Smallest deployable unit in Kubernetes.
  - Runs one or more containers.
  - Pods live on Nodes.

### 2. Control Plane

- The brain of Kubernetes.
- Manages cluster state, scheduling, scaling, and health.
- In Minikube:
  - Control plane always runs inside the first node (minikube).
  - Other nodes (minikube-m02, minikube-m03) are workers.
  - In production, control plane usually runs on separate dedicated nodes.

### 3. Project Node Mapping

**Node A (minikube)**
  - Control Plane + Worker.
  - Label: type=application.
  - Runs API Pods.

**Node B (minikube-m02)**
  - Worker only.
  - Label: type=database.
  - Runs DB Pods.

**Node C (minikube-m03)**
  - Worker only.
  - Label: type=dependent_services.
  - Runs observability/Vault Pods.

### 4. Pod Scheduling

- Kubernetes decides where Pods run using:
  - Default Scheduling
  - Scheduler picks a healthy Node with enough resources.
  - Node Labels + nodeSelector

**Label Nodes:**
```sh
kubectl label node minikube type=application
kubectl label node minikube-m02 type=database
kubectl label node minikube-m03 type=dependent_services
```

**Pod spec forces scheduling:**

```sh
spec:
  nodeSelector:
    type: database
```

#### Taints and Tolerations

- Repel Pods unless explicitly tolerated.
- Example: Only allow DB Pods on DB node.

#### Affinity/Anti-Affinity

- Run Pods on specific Nodes or separate replicas across Nodes.
- Resource Requests/Limits
- Ensure Pods only run where enough CPU/memory is available.

### 5. Key Analogy

- Cluster = City 
- Node = Building 
- Pod = Room inside the building
- Container = Worker doing the job
---


## Create a 3-node Kubernetes Cluster on 1 VM (with Minikube)

### 1. Install Requirements on your VM

- On your EC2/VM (Ubuntu/Debian assumed):

```sh
# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Install Docker (used by Minikube as a driver)
sudo apt-get install -y docker.io

# Allow non-root to use Docker
sudo usermod -aG docker $USER
newgrp docker

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube


# Verify installs:

docker --version
kubectl version --client
minikube version
```  

### 2. Start a 3-node Minikube cluster

**Run:**

```sh
minikube start --nodes 3 --driver=docker
```

- --nodes 3 → creates 3 Kubernetes nodes.
- --driver=docker → each node runs as a lightweight Docker container inside your VM.

**This will give you:**

- minikube (control-plane)
- minikube-m02 (worker)
- minikube-m03 (worker)

### 3. Verify Nodes

**Check if the cluster is ready:**

```sh
kubectl get nodes -o wide
```

**Expected output:**
```sh
NAME           STATUS   ROLES           AGE   VERSION
minikube       Ready    control-plane   Xs    v1.xx.x
minikube-m02   Ready    <none>          Xs    v1.xx.x
minikube-m03   Ready    <none>          Xs    v1.xx.x
```

### 4. Label the Nodes

**Now assign labels as per project requirement:**

```sh
kubectl label node minikube type=application
kubectl label node minikube-m02 type=database
kubectl label node minikube-m03 type=dependent_services
```

#### Check labels:
```sh
kubectl get nodes --show-labels
```
```sh
ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get nodes --show-labels
NAME           STATUS   ROLES           AGE    VERSION   LABELS
minikube       Ready    control-plane   2d8h   v1.34.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=minikube,kubernetes.io/os=linux,minikube.k8s.io/commit=65318f4cfff9c12cc87ec9eb8f4cdd57b25047f3,minikube.k8s.io/name=minikube,minikube.k8s.io/primary=true,minikube.k8s.io/updated_at=2025_09_13T08_05_55_0700,minikube.k8s.io/version=v1.37.0,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=,type=application
minikube-m02   Ready    <none>          2d8h   v1.34.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=minikube-m02,kubernetes.io/os=linux,minikube.k8s.io/commit=65318f4cfff9c12cc87ec9eb8f4cdd57b25047f3,minikube.k8s.io/name=minikube,minikube.k8s.io/primary=false,minikube.k8s.io/updated_at=2025_09_13T08_06_19_0700,minikube.k8s.io/version=v1.37.0,type=database
minikube-m03   Ready    <none>          2d8h   v1.34.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=minikube-m03,kubernetes.io/os=linux,minikube.k8s.io/commit=65318f4cfff9c12cc87ec9eb8f4cdd57b25047f3,minikube.k8s.io/name=minikube,minikube.k8s.io/primary=false,minikube.k8s.io/updated_at=2025_09_13T08_06_48_0700,minikube.k8s.io/version=v1.37.0,type=dependent_services

ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get nodes -o json | jq '.items[].metadata.labels'
{
  "beta.kubernetes.io/arch": "amd64",
  "beta.kubernetes.io/os": "linux",
  "kubernetes.io/arch": "amd64",
  "kubernetes.io/hostname": "minikube",
  "kubernetes.io/os": "linux",
  "minikube.k8s.io/commit": "65318f4cfff9c12cc87ec9eb8f4cdd57b25047f3",
  "minikube.k8s.io/name": "minikube",
  "minikube.k8s.io/primary": "true",
  "minikube.k8s.io/updated_at": "2025_09_13T08_05_55_0700",
  "minikube.k8s.io/version": "v1.37.0",
  "node-role.kubernetes.io/control-plane": "",
  "node.kubernetes.io/exclude-from-external-load-balancers": "",
  "type": "application"
}
{
  "beta.kubernetes.io/arch": "amd64",
  "beta.kubernetes.io/os": "linux",
  "kubernetes.io/arch": "amd64",
  "kubernetes.io/hostname": "minikube-m02",
  "kubernetes.io/os": "linux",
  "minikube.k8s.io/commit": "65318f4cfff9c12cc87ec9eb8f4cdd57b25047f3",
  "minikube.k8s.io/name": "minikube",
  "minikube.k8s.io/primary": "false",
  "minikube.k8s.io/updated_at": "2025_09_13T08_06_19_0700",
  "minikube.k8s.io/version": "v1.37.0",
  "type": "database"
}
{
  "beta.kubernetes.io/arch": "amd64",
  "beta.kubernetes.io/os": "linux",
  "kubernetes.io/arch": "amd64",
  "kubernetes.io/hostname": "minikube-m03",
  "kubernetes.io/os": "linux",
  "minikube.k8s.io/commit": "65318f4cfff9c12cc87ec9eb8f4cdd57b25047f3",
  "minikube.k8s.io/name": "minikube",
  "minikube.k8s.io/primary": "false",
  "minikube.k8s.io/updated_at": "2025_09_13T08_06_48_0700",
  "minikube.k8s.io/version": "v1.37.0",
  "type": "dependent_services"
}
ubuntu@ip-10-0-6-246:~/Flask-REST-API$ kubectl get nodes -o custom-columns=NAME:.metadata.name,TYPE:.metadata.labels.type
NAME           TYPE
minikube       application
minikube-m02   database
minikube-m03   dependent_services
ubuntu@ip-10-0-6-246:~/Flask-REST-API$ 
```
### 5. Test the Cluster

**Run a test pod:**

```sh
kubectl run test-pod --image=nginx --restart=Never
kubectl get pod test-pod -o wide
```

**Delete after test:**
```sh
kubectl delete pod test-pod
```
