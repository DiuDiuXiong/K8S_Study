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
    - resources:
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