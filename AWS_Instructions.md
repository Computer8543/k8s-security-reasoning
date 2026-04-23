---

## Quick Start: EKS (Amazon Elastic Kubernetes Service)

### Assumptions

- You have a AWS account
- You can create EKS clusters
- You will use **AWS CloudShell**
- You will operate from the `us-east-2` region

---

### 1. Set project and region

```bash
gcloud config set project YOUR_PROJECT_ID
gcloud config set compute/region us-central1
```

---

### 2. Create a GKE cluster

Create dedicated VPC:

```bash
gcloud compute networks create sec-dojo-vpc --subnet-mode=custom

gcloud compute networks subnets create sec-dojo-subnet \
  --network sec-dojo-vpc \
  --range 10.10.0.0/16 \
  --region us-central1
```

Create the cluster within the new VPC:

```bash
gcloud container clusters create sec-dojo-cluster \
  --num-nodes=2 \
  --enable-network-policy \
  --network sec-dojo-vpc \
  --subnetwork sec-dojo-subnet \
  --region us-central1
```

Configure kubectl:

```bash
gcloud container clusters get-credentials sec-dojo-cluster --region us-central1
```

Inspect the kubeconfig created by this command:

```bash
ls ~/.kube
cat ~/.kube/config
```

Verify cluster connectivity and explore the API surface:

```bash
kubectl cluster-info
kubectl api-resources
kubectl get namespaces
kubectl get nodes
```

---

### 3. Apply the manifest

```bash
kubectl apply -f main.yaml
```

---

### 4. Inspect Kubernetes resources

```bash
kubectl get all -n demo
kubectl get sa,role,rolebinding,networkpolicy -n demo
```

---

## Observing Cloud-Side Resources (Compute, Storage, Network)

Kubernetes declares *intent*.  
The cloud provider control plane implements *infrastructure*.

Explicitly observe resources created indirectly by Kubernetes.

---

### Compute (Nodes as VMs)

Kubernetes schedules pods onto nodes.  
The cloud provider runs **virtual machines**.

```bash
gcloud compute instances list
```

Reason about:
- How many VMs exist?
- Which zone and subnet are they in?
- What permissions do they inherit?

---

### Storage (PVCs as Disks)

PersistentVolumeClaims map to **cloud disks**.

```bash
gcloud compute disks list
```

Reason about:
- Which disks back Kubernetes PVCs?
- What happens to disks if pods or namespaces are deleted?
- What data persists beyond workload lifetime?

---

### Networking (Services as Infrastructure)

Kubernetes networking intent becomes cloud networking primitives.

```bash
gcloud compute firewall-rules list --filter="network:sec-dojo-vpc"
gcloud compute routes list --filter="network:sec-dojo-vpc"
```

Reason about:
- Which rules allow ingress?
- How broadly are they scoped?
- Which paths exist outside Kubernetes visibility?

---

## Visibility & Reachability Exercises

This section grounds the threat reasoning model in **direct observation**.

Rather than assuming behavior, you will:
- inspect logs
- execute inside running containers
- observe different network access paths
- compare control-plane vs data-plane traffic
- see how node-level visibility differs from pod-level visibility

These exercises intentionally mirror the question:

> *“If I had execution here, what could I see?”*

---

### 1. Pod-Level Visibility (Logs and Exec)

Inspect application logs from the running NGINX pod:

```bash
kubectl logs deploy/nginx -n demo
```

Then execute inside the container:

```bash
kubectl exec -it deploy/nginx -n demo -- /bin/sh
```

Notice:
- execution is real, not theoretical
- logs are scoped to the workload
- identity is inherited automatically by the pod

This is the concrete form of “attacker has execution.”

---

### 2. In-Cluster Network Reachability (BusyBox)

Create a temporary pod to test in-cluster access:

```bash
kubectl run bb --image=busybox -n demo -it --restart=Never -- sh
```

From inside the BusyBox shell:

```sh
wget -qO- http://nginx.demo.svc.cluster.local
```

This tests **data-plane networking** and NetworkPolicy behavior.

Reason about:
- what is reachable by default
- what changes when NetworkPolicy is modified
- how lateral movement might begin

---

### 3. Control-Plane Mediated Access (Port Forward)

From your local environment, forward a port to the Service:

```bash
kubectl port-forward svc/nginx -n demo 8080:80
```

Then:

```bash
curl http://localhost:8080
```

Notice:
- access succeeds even if NetworkPolicy would block pod-to-pod ingress
- kubectl is acting as a privileged control-plane proxy

This highlights the difference between **control-plane access**
and **data-plane access**.

---

### 4. Public Access Path (Cloud Load Balancer)

If the Service is of type `LoadBalancer`, retrieve the external IP:

```bash
kubectl get svc nginx -n demo
```

Once an `EXTERNAL-IP` is assigned:

```bash
curl http://<EXTERNAL-IP>
```

Compare:
- port-forward traffic path
- in-cluster traffic path
- public ingress via cloud-managed load balancer

Each path crosses **different boundaries**
and is governed by different controls.

---

## Node-Level Visibility with a DaemonSet (Advanced)

A DaemonSet runs **one pod per node**.

This exercise is not about exploitation.
It is about **making the node boundary visible**.

Inspect DaemonSet pods:

```bash
kubectl get pods -n demo -l app=node-logger
```

Exec into one of the DaemonSet pods:

```bash
kubectl exec -it <node-logger-pod> -n demo -- sh
```

Inspect the log file written on the node filesystem:

```sh
cat /var/log/node-logger/heartbeat.log
```

Reason about:
- how pod visibility differs from node visibility
- what persists beyond individual pod lifecycles
- why DaemonSets are powerful — and potentially risky

This is a concrete example of **crossing abstraction boundaries**.

## Break and Reason

Introduce failure deliberately:
- remove the NetworkPolicy
- change the Service type
- delete and recreate pods
- delete the namespace but not the cluster

Observe both Kubernetes behavior and cloud-side effects.

---

## Teardown and Verification

Teardown is part of the learning exercise.

Deleting Kubernetes resources does **not** guarantee that all cloud resources
are removed.

Verification must happen **per layer**.

---

### Step 1: Delete Kubernetes resources

```bash
kubectl delete namespace demo
```

---

### Step 2: Delete the GKE cluster

Delete cluster:

```bash
gcloud container clusters delete sec-dojo-cluster --region us-central1
```

Delete corresponding VPC:

```bash
for RULE in $(gcloud compute firewall-rules list \
  --filter="network:sec-dojo-vpc" \
  --format="value(name)"); do
  gcloud compute firewall-rules delete "$RULE"
done

gcloud compute networks subnets delete sec-dojo-subnet --region us-central1

gcloud compute networks delete sec-dojo-vpc
```

---

## Layered Teardown Verification

### Compute Verification

Ensure no cluster-related VMs remain:

```bash
gcloud compute instances list
```

Lingering instances represent:
- cost leakage
- unmanaged compute
- unexpected attack surface

---

### Storage Verification

Ensure no disks remain:

```bash
gcloud compute disks list
```

Lingering disks represent:
- persistent data exposure
- silent cost accumulation
- orphaned state outside Kubernetes control

---

### Network Verification

Ensure no networking artifacts remain:

```bash
gcloud compute firewall-rules list --filter="network:sec-dojo-vpc"
gcloud compute routes list --filter="network:sec-dojo-vpc"
```

Lingering networking resources represent:
- unintended access paths
- security group drift
- infrastructure-level exposure