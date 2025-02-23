# Core concepts

- Master Nodes: Manage, Plan, Schedule, Monitor Nodes
  - ETCD Cluster: A Key Val store for worker nodes scheduling/running/placing details (cluster stuff).
  - Kube-scheduler: Identify which worker node to put task on based on resource requirements & constraints
  - Controllers: Controls existing resources
    - Node-Controller: Control/monitor nodes
    - Replication-Controller: Desired number of containers are running all time
  - kube-apiserver: Allow all components (include worker nodes) talk to each other, most about orchestration stuff
- Worker Nodes: Host Application as Containers
  - kubelet: the kubelet (which runs on each worker node) communicates with the kube-apiserver to report node status, receive instructions about which Pods to run, and report Pod status.
  - kube-proxy: installed on each node, implements the networking part of Kubernetes Services. It ensures that when an application tries to reach a Service, the traffic gets routed to a healthy Pod that provides that service, and it does this by configuring local network rules on each node.
  - container-engine: usually docker

A **container runtime** is the software that executes containers and manages container images on a host system. It's the engine that makes containers run. Docker's is **`containerd`**, and is compatible
with K8S OCI. K8S need those to know exactly image requirement, system requirements, health status, share resources etc...
- `ctr`: CLI for containerd
- `nerdctl`: docker like CLI for containerd
- `crictl`: CRI compatible CLI from K8S for all compatible runtimes

## ETCD
The key/val store helper for K8S for orchestrating stuff, etcd also supports distributed key-val store. etcd handles the 
communication and replication directly, without Kubernetes (specifically, without components other than the API server) needing to intervene.

- [install](https://etcd.io/docs/v3.5/install/) Can also be installed as docker/pods.
- [more about ETCD]: https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/learn/lecture/19537454#content

## kube-apiserver
- `kubectl` reach `kube-apiserver` (send request to it) for different resources, can also post to it
- `kube-apiserver` is responsible for:
  - authenticate user
  - validate request
  - retrieve data
  - update etcd
  - call scheduler and get response from it (scheduler call back kube-apiserver)
  - kubelet

## Kube controller manager
This is the boss of controllers. A controller is monitoring and acting through api server for different perspective. 
- Node-Controller
  - heart beat each node 5s
  - grace period before mark as down 40s
  - eviction timeout(time for get back) 5m
  - after, remove pods and provision them in other nodes
- Replication-Controller
  - monitor amounts
- ... more controller like job, PV...

They all get composed and in a single process: `kube-controller-manager`. 

## kube-scheduler
It is only for **deciding** which job/pod goes on which node. It examines filters, resource requirements, constraints.

## `kubelet`
CLI, also refer to the helper installed on each node that talks to API server to:
- Register Node
- Create PODs
- Monitor Node & PODs

## kube proxy
kube-proxy enables network communication between Pods and Services within a Kubernetes cluster. It runs as a process on 
each node and monitors the API server for changes to Services and their corresponding Endpoints (the Pods backing the Services). 
When a Pod on a node attempts to access a Service, kube-proxy's configuration (primarily through iptables or IPVS rules 
on that node) redirects the traffic to one of the healthy Pods that implement the Service.

## Core Components
- [SRC](./Core_Components.md)