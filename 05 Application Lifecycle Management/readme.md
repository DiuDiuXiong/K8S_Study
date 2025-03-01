# Lifecycle Management

## Deployment rolling updates and rolling back
This is for the purpose of version control.

StrategyType: [Recreate | RollingUpdate]. Rolling update will scale some to new and scale down old, recreate will shut down old and recreate new.

For rolling update, check: `kubectl explain deployment.spec.strategy.rollingUpdate`
- `maxUnavailable` max below desired replicas
- `maxSurge` max above desired replicas

- `kubectl rollout status deployment/<deployment name>` to see rollout status/logs
- `kubectl rollout history deployment/<deployment name>` to see the history of different versions
  - `kubectl rollout history deployment/demo --revision=1` to see detailed definition for a specific revision
- `kubectl set image deployment/<deployment name> <container_name>=<image_name>` to change image/ or can change file and apply `kubectl apply -f <yml>`
  - `--record` will add some meaningful text to column CHANGE-CAUSE in history command, or `kubernetes.io/change-cause="Updated to version v2.0"` after apply/set for manual change reason
- `kubectl rollout undo deployment/<deployment name>` to revert last deployment
- `kubectl rollout undo deployment/<deployment name> --to-revision=x` to rollback to a specific deployment

## Configure applications

### Configure commands and arguments of applications
This is for `CMD` and `ENTRYPOINT` in docker file.
- `docker run <image_name> <new_command> <arg1> <arg2> ...` override CMD
- `docker run --entrypoint <new_executable> <image_name> <arguments_to_new_executable>` override EntryPoint
- `kubectl explain pod.spec.containers`
  - `args`: <[]string> Arguments to the entrypoint. The container image's CMD is used if this is not provided.
  - `command`: <[]string> Entrypoint array. Not executed within a shell. The container image's ENTRYPOINT is used if this is not provided.

### Configure env vars
- `pod.spec.containers.env`:
  - `name`: key name
  - `value`: corresponding value

Or we can do that via config map or secret
- `pod.spec.containers.valueFrom.[configMapKeyRef|secretKeyRef]`

To create a config map:
- `kubectl create configmap <name> --from-literal=<key>=<val> --from-literal=<key2>=<val2>`
- `kubectl create configmap <name> --from-file=<.properties file>`
- `kubectl get/describe configmaps`
- yaml: https://kubernetes.io/docs/concepts/configuration/configmap/

After creation, mount the config into pod:
- `spec.containers.envFrom` List
  - `configmapRef`:
    - `name`:...
- `spec.containers.env`: List
  - `name: ENV_KEY`:
      - `valueFrom`:
         - `configMapKeyRef`:
           `name`: config name
           `key`: key name

Or can use volume as described in yaml links.

### Configure secrets

To create secret:
- `kubectl create secrets generic --from-literal=key1=val1...`
- `kubectl create secrets generic --from-file=example.properties` (do not need to be base 64 encoded)
- yaml: https://kubernetes.io/docs/concepts/configuration/secret/, node that the value need to get based 64 encoded `echo "wuji" | base64`
- Note that when mount via volume, the secret value will be in the <mount path>/<key_name> file
- `kubectl get secrets` get secret name
- `kubectl describe secrets <secret name>` this won't show the value at all
- `kubectl get secret <secret name> -o yaml` to get secret base64 value `echo "secret abse64" | base64 --decode` to actually get the value
- To mount, either via volume as shown in the yaml, or `kubectl explain pod.spec.containers.env.valueFrom.secretKeyRef` to inject into secret or `kubectl explain pod.spec.containers.envFrom.secretRef` to inject many

Something special to node:
- secret are not secure, they are just base64 encoded with few more measures
- anyone who can create pod in a ns can create/view secrets, so need to do right RBAC
- consider third party secret providers
- A secret is only sent to a node if a pod on that node requires it.
- Kubelet stores the secret into a tmpfs so that the secret is not written to disk storage.
- Once the Pod that depends on the secret is deleted, kubelet will delete its local copy of the secret data as well.

More advanced as CKS :) coming soon

## Multi-Container Pods
These are pods that get created together and destroyed together. They share same volumes networks etc... Just add more containers under the list of `pod.spec.containers`.
Some common patterns are:
- Container + Logger (SideCar)
- Container + Adaptor (Adaptor)
- Container + Ambassador (Ambassador)
More in CKAD :)

## Init-Containers
In a multi-container pod, each container is expected to run a process that stays alive as long as the POD's lifecycle. 
For example in the multi-container pod that we talked about earlier that has a web application and logging agent, both 
the containers are expected to stay alive at all times. The process running in the log agent container is expected to 
stay alive as long as the web application is running. If any of them fails, the POD restarts.

But at times you may want to run a process that runs to completion in a container. For example a process that pulls a 
code or binary from a repository that will be used by the main web application. That is a task that will be run only 
one time when the pod is first created. Or a process that waits  for an external service or database to be up before 
the actual application starts. That's where initContainers comes in.

An initContainer is configured in a pod like all other containers, except that it is specified inside a `initContainers` section -> `pod.spec.initContainers`

When a POD is first created the initContainer is run, and the process in the initContainer must run to a completion before the real container hosting the application starts.
You can configure multiple such initContainers as well, like how we did for multi-containers pod. In that case each init container is run one at a time in sequential order.
If any of the initContainers fail to complete, Kubernetes restarts the Pod repeatedly until the Init Container succeeds.

## Auto Scaling
More advanced at: https://learn.kodekloud.com/user/courses/kubernetes-autoscaling

- Vertical Scaling: Increases resources (CPU, memory) of an existing server.
- Horizontal Scaling: Increases server count by adding more instances.

Kubernetes is specifically designed for hosting containerized applications and incorporates scaling based on current demands. It supports two main scaling types:
- Workload Scaling: Adjusting the number of containers or Pods running in the cluster.
  - Horizontal Scaling: Create additional Pods. `kubectl scale ...` or HPA (horizontal pod auto scaler)
  - Vertical Scaling: Modify the resource limits and requests for existing Pods. `kubectl edit ...` VPA (vertical pod auto scaler)
- Cluster (Infrastructure) Scaling: Adding or removing nodes (servers) from the cluster.
  - Horizontal Scaling: Add more nodes. `kubectl join ...` or cluster autoscaler
  - Vertical Scaling: Enhance the resources (CPU, memory) of existing nodes (very rare)

If we scale deployment to level such that there's no sufficient resource to support that, some parts will be created, and deployment will remain in pending.

## HPA (Horizontal Pod Autoscaler)
HPA can set some target so that K8S will create more pods if some top metrics exceed certain threshold. If it belows certain threshold, it will try to scale down to meet that.
HPA will fail if the resource field missing in the pod deployment. Note, this is reference to `requests`.


To create HPA, need to enable metric server on.

- declarative way: Yaml at: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
- imperative way: `kubectl autoscale deployment my-app --cpu-percent=50 --min=1 --max=10` (for existing deployment)
- `kubectl get hpa` to see target/current
- `kubectl delete hpa my-app` to delete hpa

## VPA
For manual methods, if we edit the requests, the default will destroy & recreate. There is an alpha for inplace resize: https://kubernetes.io/docs/concepts/workloads/autoscaling/#requirements-for-in-place-resizing.

Some drawbacks:
- Supported Resources: Only CPU and memory can be updated in place.
- QoS Class: The Pod Quality of Service (QoS) class cannot be modified through in-place resizing.
- Container Eligibility: Init containers and ephemeral containers are not eligible for resizing.
- Immutable Resource Positions: Once set, a container's resource requests and limits cannot be repositioned.
- Memory Limit Constraints: A container’s memory limit cannot be reduced below its current usage. If an update attempts to lower the memory limit too far, the resize operation will remain in progress until a permissible limit is reached.
- Platform Support: Windows Pods are not supported by the in-place resizing feature at this time.

VPA:
- `git clone https://github.com/kubernetes/autoscaler.git` && `cd autoscaler/vertical-pod-autoscaler` && `./hack/vpa-up.sh`
- VPA Recommender: Continuously monitors resource usage via the Kubernetes metrics API, aggregates historical and live data, and provides recommendations for optimal CPU and memory allocations.
- VPA Updater: Evaluates running pods against recommendations, evicting those with suboptimal resource requests.
- VPA Admission Controller: Intercepts the pod creation process and mutates pod specifications based on the recommender's suggestions, ensuring new pods start with proper resource allocations.

We then need to a configuration to let the three watch/map to some deployment. 

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "my-app"
      minAllowed:
        cpu: "250m"
      maxAllowed:
        cpu: "2"
      controlledResources: ["cpu"]
```

This is something very alpha. Note, the auto scale is based on original Pod's resource requests & `controlledResources`.

- Auto	目前是 Recreate，将来可能改为就地更新
- Recreate	VPA 会在创建 Pod 时分配资源请求，并且当请求的资源与新的建议值区别很大时通过驱逐 Pod 的方式来更新现存的 Pod
- Initial	VPA 只有在创建时分配资源请求，之后不做更改
- Off	VPA 不会自动更改 Pod 的资源需求，建议值仍会计算并可在 VPA 对象中查看

Use  `kubectl get vpa flask-app -o yaml` to get the recommended value.

| Feature                     | Vertical Pod Autoscaler (VPA)                                                              | Horizontal Pod Autoscaler (HPA)                                                                  |
|-----------------------------|--------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Scaling Method              | Adjusts CPU and memory configurations for individual pods (may require pod restarts)       | Scales the number of pods based on demand                                                       |
| Pod Behavior                | Might cause downtime due to pod restarts during resource updates                           | Typically maintains availability by adding or removing pods on the fly                           |
| Traffic Spikes              | Less effective during sudden spikes, as pod restarts are needed                            | More responsive to rapid traffic changes by dynamically adjusting pod count                      |
| Cost Optimization           | Prevents over-provisioning by matching resource allocations with actual usage             | Reduces costs by eliminating the overhead of running idle pods                                 |
| Ideal Use Cases             | Best suited for stateful workloads and resource-intensive applications (e.g., databases, JVM apps, AI workloads) | Ideal for stateless services like web applications and microservices where quick scaling is crucial |