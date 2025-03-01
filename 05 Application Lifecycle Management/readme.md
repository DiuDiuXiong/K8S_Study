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
