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