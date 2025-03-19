# Security

## Primitives

For security, basically its all about **who** can do **what**.

The entry for K8S is at `kube-apiserver` as this is the center of all communications. This can be break down into
- Authentication: Authentication verifies the identity of a user or service before granting access to the API server. Kubernetes offers various authentication mechanisms to suit different security needs:
  - Static user IDs and passwords
  - Tokens
  - Client certificates
  - Integration with external authentication providers (e.g., LDAP)
  - Service account
- Authorization: After authentication, authorization determines what actions a user or service is allowed to perform. 
  - Role-Based Access Control (RBAC) - default, associates identities with specific permissions.
  - Attribute-Based Access Control (ABAC)
  - Node Authorization
  - Webhook-based authorization
- Securing Component Communications: Secure communications between Kubernetes components are enabled via TLS encryption. This ensures that data transmitted between key components remains confidential and tamper-proof. Encryption protects:
  - Communication within the etcd cluster
  - Interactions between the Kube Controller Manager and Kube Scheduler
  - Links between worker node components such as the Kubelet and Kube Proxy
- Network Policies: By default, pods in a Kubernetes cluster communicate freely with one another. To restrict unwanted interactions and enhance security, Kubernetes provides network policies.
  - Control traffic flow between specific pods
  - Enforce security rules at the network level

## Authentication
Within a Kubernetes environment, there are different types of users:
- Human Users: Administrators and developers who perform administrative tasks and deploy applications.
- Service Accounts: Robot users that represent processes, services, or third-party applications.

Kubernetes does not manage human user accounts natively. Instead, it relies on external systems such as static files 
containing user credentials, certificates, or third-party identity services (e.g., LDAP, Kerberos) for authentication. 
Every request—whether from the kubectl command-line tool or direct API calls—is processed by the Kubernetes API server, 
which authenticates the incoming requests using one or more available authentication mechanisms.

File based is deprecated, just for learning purpose:
- `--basic-auth-file=user-details.csv` where csv in format of `password123,user1,uid0001` when start kube-apiserver, then request by `curl ... -u "username:password"`
- `--token-auth-file=user-token-details.csv` where csv in format of `KpjCVbI7cFAHYPkByTIzRb7gulcUc4B,user10,u0010,group1`, then request by `curl ... --header "Authorization: Bearer KpjCVbI7cFAHYPkByTIzRb7gulcUc4B"`
  - `kubectl get pods --token=x` to include tokens

## TLS
- Key Pair Function:
  - Public and private key pairs enable reciprocal encryption and decryption.
  - A private key encrypts a symmetric key.
  - The corresponding public key decrypts that symmetric key.
- Identity Verification via Certificate:
  - To verify that a domain, user, or identity legitimately corresponds to a claimed public key, a digital certificate is provided upon connection initiation.
- Certificate Contents and Signature:
  - The digital certificate contains:
    - The entity's public key.
    - The entity's domain or identity.
    - A cryptographic hash of the certificate's content, signed by the Certificate Authority's (CA) private key.
- Signature Decryption/Verification:
  - The CA's public key is used to verify the digital signature.
  - This confirms the certificate's integrity and authenticity.
- Chain of CA Verification:
  - If intermediate CAs are involved, their certificates are verified by their parent CAs, forming a chain.
- Root CA Trust:
  - The process continues until a root CA is reached.
  - The root CA's public key is pre-stored, establishing a foundational point of trust.

Public key/certificate (`.crt`,`.pem`), private key (`.key` or `-key.pem`)

For the scope of K8S, here's how server/client certificate works ![1.jpg](1.jpg)
And now, we need a CA for that.

To create this chain:
#### 1 Generate CA's public/private key
```bash
openssl genrsa -out ca.key 2048
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```
This key is then distributed by config/volume etc usually stored in kubeconfig file.

#### 2 Generate key for Admin users
```bash
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```
Like before, but this time use ca.key, the public key to generate its own certificate.
`curl https://kube-apiserver:6443/api/v1/pods  --key admin.key --cert admin.crt --cacert ca.crt` Then can use this to talk to others,
tbh ca.crt not necessary as api-server would have ca.key on their side and can use that key to verify admin.crt.

#### 3 Server Side certification
Etcd usually start like this.

```bash
cat etcd.yaml
- --advertise-client-urls=https://127.0.0.1:2379
- --key-file=/path-to-certs/etcdserver.key
- --cert-file=/path-to-certs/etcdserver.crt
- --client-cert-auth=true
- --data-dir=/var/lib/etcd
- --initial-advertise-peer-urls=https://127.0.0.1:2380
- --initial-cluster=master=https://127.0.0.1:2380
- --listen-client-urls=https://127.0.0.1:2379
- --listen-peer-urls=https://127.0.0.1:2380
- --name=master
- --peer-cert-file=/path-to-certs/etcdpeer1.crt
- --peer-client-cert-auth=true
- --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
- --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
- --snapshot-count=10000
- --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```
Note it pass its own public key/certificate/ca-cert:
`--key-file`/`--cert-file`/`--trusted-ca-file`

For Api server, one more step is to add alternative DNS names:
- `openssl req -new -key apiserver.key -subj "/CN=kube-apiserver" -out apiserver.csr`
- modify `SAN`
```bash
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name


[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names


[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```
Then specify keys, need to specify keys for kubelet/etcd server's public key, certificate, their root certificate's certificate.

Kubelet Certificates is like what we do for ETCD's certificate, nothing special.

To verify/see those certificates, log the pods, then locate the certificate, then:
`openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout` To decode the certificate.

### Certificate API
As the number of users increases, manually signing certificate requests becomes impractical. Kubernetes addresses this challenge with a built-in Certificates API that automates CSR management and certificate rotation.

- `openssl genrsa -out jane.key 2048` use generate a csr
- Then admin create a `CertificateSigningRequest`, with request set to base64 encode of csr `... | base64 -w 0` to return in single line.
- `kubectl get csr` -> give name section in metadata
- `kubectl certificate approve jane`
- `cluster-signing-cert-file/cluster-signing-key-file` are used by controller manager to actually perform the signing

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  expirationSeconds: 600 # seconds
  usages:
    - digital signature
    - key encipherment
    - server auth
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS
```
```kubectl explain certificatesigningrequest```

## Config
Yaml: https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/

Previously, we need `--cacert`, `--key`, `--cert`, `--endpoint` for authentication. It is stored in `~/.kube/config`. It contains three part
- `clusters`:
  - `cluster`:
    - `certificate-authority|certificate-authority-data`: the cert of the cluster api server
    - `server: https://127.0.0.1:49722` server point
  - `name`: name for cluster
- `users`:
  - `name`: name of user
  - `user`:
    - `client-certificate: /Users/wujizhu/.minikube/profiles/minikube/client.crt`
    - `client-key: /Users/wujizhu/.minikube/profiles/minikube/client.key`
- `contexts`: mapping between users and clusters
  - `context`:
    - `cluster`: cluster name
    - `namespace`: default
    - `user`: name of user
  - `name`: name of context itself

- `kubectl get config` to view configs
- `kubectl config current-context`/ `kubectl config set-context ...` to see switch context.
This way we don't need to include all the data.

## Authorization
This is when we pass sufficient keys, now its time for what can we do. 

### API groups
The Kubernetes API is the primary interface for interacting with your cluster. Whether using the command-line tool 
kubectl or directly sending HTTP requests via REST, every interaction communicates with the API server.

![img.png](img.png)
![img_1.png](img_1.png)
![img_3.png](img_3.png)

`kubectl proxy` to start a proxy for your http request to kubeapi server by using the kubeconfig file.

### Authorization
There are four types:
- Node Authorization
- Attribute-Based Authorization
- Role-Based Access Control (RBAC)
- Webhook Authorization
- AlwaysAllow
- AlwaysDeny

Node authorizer are for requests to node resources.
![img_4.png](img_4.png)

Attribute-based authorization associates specific users or groups with a defined set of permissions. For example, you 
can grant a user called "dev-user" permissions to view, create, and delete pods. This is achieved by creating a policy 
file in JSON format and passing it to the API server. Consider the following example policy file:

```
{"kind": "Policy", "spec": {"user": "dev-user", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"user": "dev-user-2", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"group": "dev-users", "namespace": "*", "resource": "pods", "apiGroup": "*"}}
{"kind": "Policy", "spec": {"user": "security-1", "namespace": "*", "resource": "csr", "apiGroup": "*"}}
```
Each time security requirements change, you must manually update this policy file and restart the Kube API Server. 
This manual process can be tedious and set the stage for more streamlined methods such as Role-Based Access Control (RBAC).

RBAC is considered the standard method for managing access within a Kubernetes cluster. The diagram below provides a 
visual representation of RBAC across different roles:
![img_5.png](img_5.png)

External Authorization Mechanisms connects with external apis for access control.
`...api-server-binary --authorization-mode=Node,RBAC,Webhook`

When a user’s request is first evaluated by the node authorizer. If the request does not pertain to node-specific 
actions and is consequently denied, it is then passed to the next module, such as RBAC. Once a module approves the 
request, further checks are bypassed and the user is granted access.

### RBAC
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["ConfigMap"]
  verbs: ["create"]
```
If it is part of core api group, we can leave apiGroups empty, otherwise, we need to specify it. It can have multiple list of rules. Can also specify namespace in metadata section.
Then we create a binding:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```
Note that subjects are who owns the role, and role ref only refer to one role.
- `kubectl get roles`/ `kubectl get rolebindings`/ `kubectl describe role developer`/ `kubectl describe rolebinding devuser-developer-binding`
- `kubectl auth can-i create deployments` / `kubectl auth can-i create deployments --as dev-user` to verify if I can do something
- `kubectl create role/rolebinding --help` for short cuts

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "create", "update"]
  resourceNames: ["blue", "orange"]
```
Can even filter on top of name.

### Cluster Roles
There are some resources not under the scope of namespace, instead they are under whole cluster level, like nodes, PVs
![img_6.png](img_6.png)
We need to use `ClusterRole` and `ClusterRoleBinding` for this.
- `kubectl api-resources --namespaced=true`
- `kubectl api-resources --namespaced=false`
For resource that are within namespace scope and not that scope.

The yaml format is pretty similar to roles.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

It's important to note that while cluster roles and role bindings are primarily used for cluster-scoped resources, they 
can also manage access to namespace-scoped resources. When you bind a cluster role that grants permissions on pods, 
for instance, the user will have access to pods in every namespace—unlike a namespaced role which restricts access to 
a single namespace.

### Service accounts
Its just another format of user, it is for service bots. Before when creating a service account, a token will automatically create for that, 
now need to do that manually. Also, now when create a service account, can bind roles to it just like users.
- `kubectl create serviceaccount dashboard-sa` to create
- `kubectl create token dashboard-sa` to give it a token
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  serviceAccountName: dashboard-sa
  containers:
  - name: my-kubernetes-dashboard
    image: my-kubernetes-dashboard
```
To add the account permission to a pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  automountServiceAccountToken: false
  containers:
  - name: my-kubernetes-dashboard
    image: my-kubernetes-dashboard
```
TO avoid default service account.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: my-namespace
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: my-namespace
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
To bind additional roles to service account.

### Secure Image
When pull from private repository, instead of `docker login`, we should:
- In the image of yaml file of pod, specify the full path
- Create a docker-registry type of secret: https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_secret_docker-registry/.
- `kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \
  --docker-username=registry-user \
  --docker-password=registry-password \
  --docker-email=registry-user@org.com` 
- Include it in `imagePullSecrets: -name`

### Security Context
Security context is same as docker capabilities. By default, docker has `mnt`, `uts`, `ipc`, `pid`, `net`. We need --cap-add or --cap-drop.

Security context for k8S is same as that, it can set at pod or container level or both. Pod one will overwrite container one.
- `runAsUser` to override user
- `capabilities` for --cap-add or --cap-drop

### Network Policies
Ingress means traffic coming into the pod, egress means traffic coming outside pod. If pod A is sending request to pod B, then egress of pod A is ingress of pod B.

To define network policy: see yaml: https://kubernetes.io/docs/concepts/services-networking/network-policies/ Nothing special. Note that if we don't put a block,
then it means default no constraints, if we put an empty skeleton block, it means deny everything.
- `podSelector` to define what pod to put the policy onto
- `from` to define what pod can go through such defined network policy.
- `namespaceSelector` to scope pods within a ns.
- `ipBlock` allows for a specific range of ips
- for things within the from/to list, each element holds a or relationship, while within the same list item, all sub feature holds an and relationship.

https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/learn/lecture/27430646#content some bonus for quick context switch.

### Custom Resource Definitions (CRD)
This means some random block in yaml that is not originally part of K8S's definition.
https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/
We need to first apply the custom resource definition then will be able to create it.

To interact with it, we will need custom controller and operator as well.