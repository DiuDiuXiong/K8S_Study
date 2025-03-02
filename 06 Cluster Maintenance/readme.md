# Cluster Maintenance

## OS upgrade
Sometime we need to shut down some nodes for temporarily maintenance. If the Pod on that node is part of a rs, then it will
be rescheduled on others, otherwise, we just lost it.

- If a node comes back online quickly, the pods are simply restarted.
- If a node remains down for more than five minutes, Kubernetes marks the pods on that node as dead. rs will recreate, other will just lost.
  - `kube-controller-manager --pod-eviction-timeout=5m0s...`
- `kubectl drain node-1` To evict Pods on a node to other node + mark the node as unschedulable
- `kubectl uncordon node-1` To mark the node as schedulable
- `kubectl cordon node-1` To mark the node as not schedulable

If try to drain a node that has pod not part of a rs, it will errored, unless we force.

## Cluster Upgrade (K8S version upgrade)
When we `kubectl get nodes`, we see there's version column on nodes: e.g. v1.13.2.
- <Major>.<Minor>.<Patch>
- Major versions indicate significant changes.
- Minor versions are released every few months and introduce new features and functionalities.
- Patch versions are released more frequently to address critical bug fixes.

- Alpha releases: Features are disabled by default and may be unstable.
- Beta releases: Features are enabled by default and undergo thorough testing.
- Stable releases: Fully tested and ready for production use.

Note that while the control plane components share the same version, additional components like the ETCD cluster and CoreDNS servers may have their own version numbers because they are maintained as separate projects.

It is important to note that not all components are required to run on the same version. Although different components can operate on varying release versions, the Kube API Server remains the primary control plane component that all others communicate with. Consequently, no component should ever run on a version higher than the API Server. For example:
- The controller manager and scheduler may be one version lower than the API Server.
- The Kubelet and Kube Proxy components may be two versions lower than the API Server.

### When To Upgrade
Suppose you're running Kubernetes 1.10, and new releases 1.11 and 1.12 are available. Kubernetes officially supports up to the three most recent minor versions. With 1.12 as the latest, the supported versions are 1.12, 1.11, and 1.10. When version 1.13 is released, only 1.13, 1.12, and 1.11 will be supported. It is advisable to upgrade your cluster to the next release before support for your current version is dropped.

### Upgrade Strategy
An effective upgrade strategy is to upgrade one minor version at a time (e.g., upgrade from 1.10 to 1.11, then from 1.11 to 1.12, and finally from 1.12 to 1.13) rather than attempting a large jump between versions. Keep in mind that the upgrade process may vary depending on your cluster setup. Managed Kubernetes services (such as Google Kubernetes Engine) offer a simple upgrade interface, while clusters deployed using tools like kubeadm or manual installation require more hands-on management.

### Upgrade Process Overview
- Upgrading the master nodes: During the master node upgrade, control plane components (such as the API server, scheduler, and controller managers) experience a brief interruption.
- Upgrading the worker nodes:
  - Upgrade all worker nodes simultaneously (which may result in downtime).
  - Upgrade one worker node at a time, allowing workloads to be shifted and ensuring continuous service.
  - Add new nodes with the updated software version, migrate workloads to these new nodes, and then decommission the older nodes.

### Upgrading with kubeadm
1. Upgrade the kubeadm tool version:
   - `apt-get upgrade -y kubeadm=1.12.0-00`
2. Upgrade other components on master:
   - `kubeadm upgrade apply v1.12.0` (`kubeadm upgrade plan` for possible versions)
3. Upgrading Worker Nodes
   - `kubectl drain node-1`
   - upgrade kubeadm + restart the kubelet service + made node as schedulable

## Demo Upgrade
Reference: https://kubernetes.io/docs/tasks/administer-cluster/cluster-upgrade/, `kubeadm` source: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

- Updating package repositories to the new packages.k8s.io.
- Upgrading kubeadm on the control plane and verifying the available versions.
  - `sudo apt update && sudo apt-cache madison kubeadm` to see newest possible version
- Running a dry-run upgrade plan to check compatibility.
- Applying the control plane upgrade.
  - `sudo apt-mark unhold kubeadm && sudo apt-get update && sudo apt-get install -y kubeadm='1.32.x-*' && sudo apt-mark hold kubeadm`
- Upgrading kubelet and kubectl on both the control plane and worker nodes while minimizing downtime by draining and uncordoning nodes.
  - `sudo apt-mark unhold kubelet kubectl && sudo apt-get update && sudo apt-get install -y kubelet='1.32.x-*' kubectl='1.32.x-*' && sudo apt-mark hold kubelet kubectl`

Note that some control plane may only have control components but not kubelet & kubectl.

## Backup and Restore
For most Kubernetes deployments, consider backing up:
- Declarative Configuration Files: Files defining resources like Deployments, Pods, and Services.
- Cluster State: Information stored in the etcd cluster.
- Imperative Objects: Resources created on the fly (e.g., namespaces, secrets, configMaps) which might not be documented in files.

### Declarative Backup
Using a declarative approach — creating definition files and applying them with kubectl — not only documents your configuration but also makes it reusable and shareable. Can have some version control around that.

### Imperative Backup
`kubectl get all --all-namespaces -o yaml > all-deploy-services.yaml`

### Backing Up the etcd Cluster
- `--data-dir=/var/lib/etcd` option when start etcd says where data is stored, can find in manifest file `/etc/kubernetes/manifests/etcd.yaml`
- Source: https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster
  - First need to export API as 3: `export ETCDCTL_API=3`
  - `etcdctl snapshot save -h` to save etcd stuff
  - `etcdctl snapshot restore -h` to restore etcd stuff
  - `--cacert` & `--cert` & `--endpoints=[127.0.0.1:2379]` & `--key` are mandatory if we are connecting to a different place, if restore file are on same machine, then don't need to specify key.

- `etcdctl snapshot save snapshot.db` to save the snapshot
- `etcdctl snapshot status snapshot.db` to check status

Then to restore
- stop all API server instances: `service kube-apiserver stop`/ `systemctl stop kube-apiserver`
- `etcdctl --data-dir <data-dir-location> snapshot restore snapshot.db` where <data-dir-location> is a directory that will be created during the restore process.
- Update etcd Configuration: Modify your etcd configuration file to point to the new data directory. (modify the --data-dir in `etcd.service` file or volume file for `/etc/kubernetes/manifest/etcd.yaml`)
- `systemctl daemon-reload`
- `service etcd restart`/ `systemctl restart etcd` or can done by just delete pods, they are in manifest, will get recreated soon
- `service kube-apiserver start`/ `systemctl start kube-apiserver` can done by just delete pods, they are in manifest, will get recreated soon

Note, the process of save/restore, is to write file in a .db file, but that's not the raw file format that will get used by etcd, so when restore, we need to give --data-dir which is where the working file will get output to.
And after that, modify etcd manifest file to point to that, which will cause pod restart and should work.