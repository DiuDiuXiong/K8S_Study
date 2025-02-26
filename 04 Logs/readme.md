# Logs

## Metric Server
`cAdvisor` is the component within `kubelet` that receives information about monitoring and send to monitor server. 
Metric server is in-memory provisioning of clusters, so no historical data, just current status like cpu usage etc. Other options are DD, El-Search etc.

- `minikube addons enable metrics-server` or clone metric server+deploy or `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml` 
- `kubectl top [nodes|pod]` to get details (need metric server up)


## Application Logs
- `kubectl logs -f <pod name>` just like what we did for docker
- `kubectl logs -f <pod name> <image name>` if there are more than one containers

