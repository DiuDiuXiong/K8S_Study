# Core Components

## Yaml in K8S
Its used to define more complex K8S components. Always contain/require:
- `apiVersion`
- `kind`
- `metadata`
- `spec`

For all workload resources, refer to: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/
- `kubectl apply -f <yaml file>` to create resources

## Pods
The smallest units of application inside K8S. We create new pods for orchestration.
- A pod can be a set of containers that work together. 
- It usually has a one-to-one relationship with a specific container.
- If we want to scale, we scale pods.
- Commands:
  - `kubectl run <pod name> --image <image name>` it first create a pod then create the image within the pod
  - `kubectl run <pod name> --image <image name> --dry-run=client -o yaml > <yaml path>` output the config yaml to a file
  - `kubectl get pods` list pods
  - `kubectl get pods -o wide` can see which node it is one in simpler way
  - `kubectl describe pod <pod name>` for details, can see container status and logs at event section
  - `kubectl delete pod <pod name> ... <more pod names>` to delete
  - yaml: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/
    - [example](./POD.yaml)
    - `metadata`: `name` for name, `label`: for key-val labels
    - `spec`: 
      - `containers`: 
        - `- name/image` use - indicate list for list of containers within pods

## ReplicaSets/ ReplicaController
Bring new ones if old ones failed, to reach desired number of Pods or auto-scale to reach loads.
Replication Controller is the old version of Replica Set.

- `kubectl create -f <definition yaml>`
- `kubectl get replicaset` to get
- `apiVersion/kind/metadata` like before
- `spec`: for what's inside
  - `template`: include everything that's in Pods definition despite apiVersion/kind
  - `replicas`: number of desired number
  - `selector` # this is important so that replicaset don't need to scan all pods
    - `matchLabels`: key/val for label selector of pods, will include existing pods as well
- To scale:
  - Change `replicas` and `kubectl replace -f replicaset-definition.yaml`
  - `kubectl scale --replicas=x -f replicaset-definition.yml` (this will not change the yaml file)
  - based on load (coming later)
- To change:
  - `kubectl edit replicaset <name>` and after edit changes will apply automatically
- To output:
  - `kubectl get replicaset/rs <replicaset name> -o yaml > <yaml path>` to get yaml for existing replicaset

## Deployments
To roll forward and roll back codes, its generally rolling out. (more info coming soon...)

- it automatically create a replica set
- nearly everything is same with replicaset despite `kind` goes to `Deployment`
- yaml: https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/
- `kubectl get all` for all resources
- `kubectl create deployment --image=nginx nginx` create deployment
- `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml` for yaml files
- `kubectl create -f nginx-deployment.yaml` to create after some edits
- `kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > nginx-deployment.yaml` to specify replica numbers

## Services
An API gateway, allows replica set/pods talk to each other, and connect to outside users.

- NodePort: Create a port on each node (based on selector), so external request to any node on that port get forwarded to one of the pod
- ClusterIP: Create a port on internal side, for pod to pod (internal service) communication
- LoadBalancer: Connect with external load balancer from cloud provider, get public address and forward to underlying NodePort or ClusterIP
- `TargetPort`: port that Pods expose
- `Port`: port that Services expose
- `NodePort`: 30000 - 32767, port that Node expose

### NodePort
Everything goes to port automatically forward to nodePort, everything goes to nodePort goes to port, hence targetPort.

- `apiVersion/kind/metadata` same see: https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/
- `spec`:
  - `type`: `NodePort`
  - `ports`: This is an array.
    - `targetPort`: 80
    - `port`: 80
    - `nodePort`: 30008
  - `selector`: Just like replica set/deployment to identify pods, those pods may get managed by replicaset/deployments as well.
  - `externalTrafficPolicy: Local`: This setting tells Kubernetes to only forward traffic to Pods that are running on the same node that received the traffic.

To create: `kubectl create -f <yml>`, then curl `<node_ip for any node>:nodePort` would able to get the service.

### ClusterIP
We do not need to refer to each other by service IP, as underlying node may change over time. We refer by service name.
There will be a default service created by K8S at launch for each namespace or when namespace get created.

- `apiVersion/kind/metadata` as before
- `spec`:
  - `type`: `ClusterIP`
  - `ports`: # list again, don't need nodePort here.
    - `targetPort`: 80
    - `port`: 80
  - `selector`: ... select pods for this service

Then we can `curl http://<service name in metadata>:<port>` to call.

### Load Balancer
Usually for frontend applications, its unrealistic to give IP of nodes to external users. We want something like `http://example.com` to
as a single entry point.

- `apiVersion/kind/metadata` as before
- `spec`:
  - `type`: `LoadBalancer`
  - `ports`:
    - `targetPort`: 80
    - `port`: 80
    - `nodePort`: 30008

For configuration, coming soon...

### Namespaces
What: A virtual cluster within a Kubernetes cluster. Think of it like folders for organizing resources (Pods, Services, etc.).

Why:
- Organization: Keeps resources logically separated.
- Isolation: Prevents naming conflicts and provides some access control.
- Resource Management: Limits resource usage within a namespace.
- Multi-tenancy: Allows multiple teams/projects to share a cluster safely.

Some features of namespace
1. Within same namespace, resources can refer to each other by names.
2. To refer resource outside namespace: `<service-name>.<namespace>.<service>.<domain>`, default is `<service-name>.<namespace>.svc.cluster.local`
3. `kubectl get pods --namespace=kube-system` to get stuff from other namespace
4. `kubectl create -f <yml> --namespace=<namespace>` or add `namespace`: `<namespace>` under metadata

Yaml: https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/namespace-v1/
- `apiVersion/kind`
- `metadata`:
  - `name`: <name of namespace>

- Or to create `kubectl create namespace <name>`
- To switch default namespace: `kubectl config set-context $(kubectl config current-context) --namepsace=<x>` to switch.
- To get stuff for all spaces: `--all-namespaces/-A` attach to the command.
- To create resource quota for a namespace:
  - yml: https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/
  - `metadata`:
    - `name`
    - `namespace`: to match
    - `spec`: for limitations

### Imperative vs Declarative
- imperative: exact about how to perform on each step, also changes in this way is not recorded
  - kubectl is imperative
    - `kubectl run/create <name of stuff> <identifier> ...`
    - `kubectl edit <name of stuff> <identifier>`
    - `kubectl replace --force -f <yml>`
    - `kubectl create -f <yml>`: raise if stuff already exist
    - `kubectl replace -f <yml>`: raise if stuff not exist
- declarative: like tf, only specify final target
  - yaml is declarative
    - `kubectl apply -f <yml>` <-, this is declarative, it will always try to figure out how to reach there instead of raise for commands above

Some imperative tricks, more for save time in CKA:
- use the `--dry-run=client` option. This will not create the resource, instead, tell you whether the resource can be created and if your command is right.
- `-o yaml`: This will output the resource definition in YAML format on screen.
- `kubectl run nginx --image=nginx`: Create an NGINX Pod
- `kubectl run nginx --image=nginx --label="key=val,key2=val2" --dry-run=client -o yaml` :Generate POD Manifest YAML file (-o yaml). Don't create it(--dry-run)
- `kubectl run nginx --image=nginx --label="key=val,key2=val2" --expose=true --port=80 --dry-run=client -o yaml` automatically create a service with same name as pod and do a 80-80 mapping
- `kubectl create deployment --image=nginx nginx` :Create a deployment
- `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml`: Generate Deployment YAML file (-o yaml). Don't create it(--dry-run)
- `kubectl create deployment nginx --image=nginx --replicas=4`: Generate Deployment with 4 Replicas
- `kubectl scale deployment nginx --replicas=4`: you can also scale a deployment using the kubectl scale command.
- `kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml`: Another way to do this is to save the YAML definition to a file and modify
- `kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml`: Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379, Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379
- `kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml`: This will not use the pods labels as selectors, instead it will assume selectors as app=redis. You cannot pass in selectors as an option. So it does not work very well if your pod has a different label set. So generate the file and modify the selectors before creating the service).
- `kubectl expose pod nginx --type=NodePort --port=80 --name=nginx-service --dry-run=client -o yaml`: (This will automatically use the pod's labels as selectors, but you cannot specify the node port. You have to generate a definition file and then add the node port in manually before creating the service with the pod.)
- `kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml`: This will not use the pods labels as selectors

## kubectl apply
It can be used to create, update stuff, and it will figure out like tf-apply.

When we apply:
- It first get converted to json **last applied configuration**, stored in k8s live object
- It modified live object within k8s
- Whenever we apply, if first compare yam file with live object, modifies difference, apply, then finally change **last applied configuration**