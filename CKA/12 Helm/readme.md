# Helm
Helm is:
- A package manager for Kubernetes:
  - It simplifies the deployment and management of applications on Kubernetes.
  - It allows you to bundle Kubernetes resources into reusable packages called "charts.
- A tool for automating Kubernetes application lifecycle:
  - It streamlines the processes of installing, upgrading, and uninstalling Kubernetes applications.
  - It helps manage complex Kubernetes deployments by organizing configurations.
- A way to manage Kubernetes manifests:
  - It helps to organize and manage the many YAML files that are needed to deploy applications to kubernetes.
  - Essentially, Helm makes working with Kubernetes much easier by providing a standardized way to package and deploy applications.

To install Helm: https://helm.sh/docs/intro/install/

## Helm components
- The Helm CLI is the command-line tool installed on your local machine. It enables you to perform actions such as installing, upgrading, and rolling back applications.
- A Helm chart is a collection of files that contains all the instructions needed to create the Kubernetes objects required by your application.

Helm stores metadata not on your local system but within your Kubernetes cluster as Kubernetes secrets. This ensures that metadata stays accessible to anyone working with the cluster and persists through cluster restarts.

### Charts and Templating
Helm charts are packages that include several resource definition files, such as templates for Deployments, Services, 
and more. The templating mechanism allows you to separate configuration values (for example, those provided in a 
values.yaml file) from the resource definitions.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: hello-world
```
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: nginx
          image: "{{ .Values.image.repository }}"
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```
```yaml
# values.yaml
replicaCount: 1
image:
  repository: nginx
```

Many public repositories offer Helm charts, but you might need to adjust the values.yaml file to tailor the installation to your specific requirements.
Below is an example of an advanced templating approach in a Helm chart:

```yaml
# deployment-advanced.yaml
apiVersion: {{ include "common.capabilities.deployment.apiVersion" . }}
kind: Deployment
metadata:
  name: {{ include "common.names.fullname" . }}
  namespace: {{ .Release.Namespace | quote }}
  labels:
    {{- include "common.labels.standard" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" (dict "value" .Values.commonLabels "context" $) | nindent 4 }}
    {{- end }}
    {{- if .Values.commonAnnotations }}
    annotations: {{- include "common.tplvalues.render" (dict "value" .Values.commonAnnotations "context" $) | nindent 6 }}
    {{- end }}
spec:
  selector:
    matchLabels: {{- include "common.labels.matchLabels" . | nindent 6 }}
  {{- if .Values.updateStrategy }}
  strategy: {{- toYaml .Values.updateStrategy | nindent 4 }}
  {{- end }}
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
```

### Release and Versioning
Every deployment of a Helm chart results in a new release. Releases allow you to run multiple instances of the same 
chart concurrently, with each release maintaining its own revision history. This functionality is especially useful 
when you need separate releases for production and development environments, even though they may be based on the same chart.

```bash
# helm install [release-name] [chart-name]
$ helm install my-site bitnami/wordpress
$ helm install my-second-site bitnami/wordpress
```

### Helm Repositories and Artifact Hub
Helm charts are available across numerous public repositories managed by providers such as Appscode, Community 
Operators, TrueCharts, and Bitnami. Instead of searching through each repository separately, Artifact Hub aggregates 
these resources in one convenient location.

### Helm directory structure
Helm charts are organized in a standard directory structure. At a minimum, a chart directory includes:
- templates directory: Contains all the template files (e.g., Service and Deployment manifests).
- values.yaml file: Provides the default configuration values.
- Chart.yaml file: Holds metadata about the chart.
- Additional files such as a LICENSE or README, and directories like charts (for dependent charts) might also be included.

### Heml commands
https://helm.sh/docs/intro/cheatsheet/

To modify values, its like tfvar files in terraform. Do one of the following
- `helm install <name> <chart> --set key1=val1,key2=val2`
- `helm install <name> <chart> --values <yaml-file/url>  # Install the chart with your specified values`
- edit the `value.yaml` file

Process all files: Helm will attempt to render every file within the templates/ directory as a Kubernetes manifest. It will try to 
create things in order: https://helm.sh/docs/intro/using_helm/

### Helm rolling forward and backward
https://helm.sh/docs/intro/cheatsheet/
`helm package ./my-custom-app` with new version will allow a new update.