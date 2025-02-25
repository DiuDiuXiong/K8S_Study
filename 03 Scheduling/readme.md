# Scheduling
Its for deciding which node pod should go or not.

## Manual scheduling
Very rare, almost never used. If there's no scheduler, then the POD will remain in `PENDING` state. To verify that:
- `kubectl describe pod <pod name>` to make sure nothing in events, its just pending
- `kubectl get pod -n kube-system` to check whether there's a scheduler defined

To manually schedule:
- `spec`:
  - `containers`: ...
  - `nodeName`: `<nodeName>` to schedule the task on a node

Note that after a pod is running on a node, we cannot change it or say cannot shift it to another node, we have to delete
and re-create it if we want to move by modifying the yaml and `kubectl replace --force -f <yml>`. 
Or we can create a binding object and send a post request to the kube-apiserver. (this method just exist, not recommend)

## Labels and Selectors

We can set labels in key val pair for pods definition like:
```yaml
metadata:
  labels:
    key: var
    key2: var2
  annotations:
    key: var
    key2: var2
```
Or when we create pod: `kubectl run nginx-dev --image=nginx --labels=version=1,env=dev`.

Then to select, we can do that in:
- command: `kubectl get pods --selector key=val,key2=val2...` or `kubectl get pods --selector key=val,key2=val2... --no-headers | wc -l` to get count, note this include headers.
- replica set/deployment: `spec.selector.matchLabels` have key vals
- service: `spec.selector` has key vals which map to those pods

Annotations are just non-selecting information more for managing purpose like app version, git, descriptions etc...

## Taints & Tolerations
Taints are applied to nodes. They prevent pods from being scheduled onto the node unless the pod has a matching toleration. Think of taints as "repelling" pods.
To set taints: `kubectl taint node <node name> key=val:taint-effect`, to remove `kubectl taint node <node name> key-`
- `NoSchedule`: Do not evict current one, but won't schedule new ones onto it
- `PreferNoSchedule`: Will give priority away from the node, no guarantee.
- `NoExecute`: Will evict current one + no new schedule onto it.

To add toleration, so that pod can be scheduled on those nodes (not required):
```yaml
spec:
  containers: ...
  tolerations:
    - key: "<key name>"
      operator: "Equal|Exists"
      value: "value"
      effect: "NoSchedule|NoExecute|PreferNoSchedule"
```

Value is only required if it is equal operator, exist only check if key exist. This will escape the effect.
`kubectl describe node node01 | grep Taints` to get taints.

## Node Selector & Node Affinity
We can add labels to nodes and select on nodes. Note this is compulsory, not like taints that Pod may get scheduled on other place.

To add labels to node: `kubectl label node <node name> [key=val|key-]`, then it can be selected in Pod by:
```yaml
spec:
  containers: ...
  nodeSelector:
    key:val
```
For more flexible selection, use affinity: `kubectl explain pod.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms.matchExpressions`
- `preferredDuringSchedulingIgnoredDuringExecution` just as its name described
- `requiredDuringSchedulingIgnoredDuringExecution` will only avoid during schedule time
- `requiredDuringSchedulingRequiredDuringExecution` planned to get added soon

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
             - key: k
               operator: [DoesNotExist|Exists|Gt|In|Lt|NotIn]
               values: An array of string values. If the operator is In or NotIn, the values array must be non-empty. If the operator is Exists or DoesNotExist, the values array must be empty. If the operator is Gt or Lt, the values array must have a single element, which will be interpreted as an integer. This array is replaced during a strategic merge patch.
```

Note that taints exclude undesired node & selector/affinity make sure pod only placed on desired node, (taints node select pod, affinity pod select node). In combination we can placec them exactly where they want.

## Requests vs Limits
`kubectl explain pod.spec.containers.resources`.
Requests is lower hard limit, limit is soft upper limit. If cpu exceed limit, it will get throttled, if memory exceed limit constantly, it will get OOMKilled and terminated (go to describe -> last state). It is important we set constraint, otherwise docker will consume as much as possible.

It is most important to have requests, so that app work as expected, it is a guarantee. Limit is more for operation purpose.

```yaml
spec:
  containers:
    resources:
      requests:
        memory: "64Mi"  # 64 Mebibytes (recommended)
        cpu: "250m"
      limits:
        memory: "1Gi"   # 1 Gibibyte (recommended)
        cpu: "1"
```

Or we can use `LimitRange`: https://kubernetes.io/docs/concepts/policy/limit-range/, these values will get filled default to Pod, so remember to include set request & limit if request > default limit. `kubectl apply -f limit-range.yaml -n <your-namespace>`
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
  - default: # this section defines default limits
      cpu: 500m
    defaultRequest: # this section defines default requests
      cpu: 500m
    max: # max and min define the limit range
      cpu: "1"
    min:
      cpu: 100m
    type: Container
```
Note that max/min is a strict range. We can also constrain the total consumed by a namespace by resource quota: https://kubernetes.io/docs/concepts/policy/resource-quotas/

You cannot edit the environment variables, service accounts, resource limits (all of which we will discuss later) of a running pod. With Deployments you can easily edit any field/property of the POD template. Since the pod template is a child of the deployment specification,  with every change the deployment will automatically delete and create a new pod with the new changes.

## DaemonSet
One per node, whenever a new node added, a daemon set automatically added to that node. Something like anti-virus or monitoring agents. Or like kube-proxy/weave-net.

Yaml: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/, everything is same as replicaset, despite kind.

- `kubectl get daemonsets <...>`
- This is done by node-affinity.

## Static Pods
This is the case that worker node completely lost master. By placing pod definition in `/etc/kubernetes/manifests`, it periodically scan the folder and create pods. (we can only do this to pods). This is called static pod. Config is usually at: `/var/lib/kubelet/config.yaml`

- path is set by: `--pod-manifest-path=` when start `kubelet`
- or `staticPodPath` and share it in `--config=<yml>`
- since we don't have api server, we should use `docker ps` to check
- when we create a static pod, there will be a mirror record in master, but we cannot modify or delete
- its mostly used for control-plane itself, and can create those pods under kubectl-system, which does not rely on existing kube-api server
- `kubectl get pods <podname> -o yaml` -> check ownerReferences, if it is node, then it is a static pod.

## Multiple Scheduler
We can have more than one scheduler, in the end, it will just start as another pod in kube-system, see reference at: https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/ Need to define config file and reference that when start the scheduler.
- Set up the authentication
- define the: `my-scheduler-config.yaml`
- then in the `config-map.yaml`, have `profiles: - schedulerName: my-scheduler`, there can be more than one profiles. And can disable/enable multiple plugins there.
- then apply Deployment
- then in pod: `spec.schedulerName` to map to the name, if name does not exist, will wait forever.

We can then verify by:
- `kubectl get pods -A` to see if there are more than one named scheduler
- `kubectl get events -o wide` to see the source, the source might be other than default scheduler
- `kubectl log <scheduler name> -n=kube-system` can see details for that scheduler

## Admission Controllers
Role-Based Access Control (RBAC) determines who can perform what actions on Kubernetes resources (like creating, deleting, 
or modifying pods, services, etc.) via the API server. Admission controllers, on the other hand, act as gatekeepers that 
can intercept and potentially modify or reject requests to the API server before they are persisted, allowing for 
fine-grained control over the content and validity of those requests, going beyond simple permission checks.

- `kube-apiserver -h | grep enable-admission-plugins` need to run in control plane node

Src: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
- To turn on: `kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...`
- To turn off: `kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny ...`
- `ps aux | grep kube-apiserver` to see options, `kube-apiserver` may not always stand as an executable. We need to exec into api server to run those commands.
  - `kubectl exec -it <pod name> -n kube-system <kube-apiserver command>` :)
  - `/etc/kubernetes/manifests/kube-apiserver.yaml` for more details, this is a static pod so can edit and wait for api-server restart.

- The `NamespaceLifecycle` admission controller will make sure that requests to a non-existent namespace is rejected and that the default namespaces such as default, kube-system and kube-public cannot be deleted.
- `NamespaceAutoProvision` create n if not exist before

## Validating and Mutating Admission Controllers
- Validating Admission Controller: Check & Validate some operations
- Mutating Admission Controller: (e.g. DefaultStorageClass) that do some change to objects before its creation
- There are some admission controllers can do both.
- Mutating before Validating, so its change can also be validated.

We can define our own configuration rules, maybe we don't want pod name same as username.
1. Define the code: https://github.com/gkampitakis/k8s-dac-demo/blob/main/webhook-server/main.go
2. Create required TLS secrets
3. Make the code as a deployment & a service pointing to that
4. https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/ have either `ValidatingWebhookConfiguration` or `MutatingWebhookConfiguration`, and point to the service. Can then add rules there to select which resource will get directed there.