![Kubernetes cluster](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

## Core Components

### Control Plane Components
- kube-apiserver: The core component server that exposes the Kubernetes HTTP API
- etcd: Consistent and highly-available key value store for all API server data
- kube-scheduler: Looks for Pods not yet bound to a node, and assigns each Pod to a suitable node.
- kube-controller-manager: Runs controllers to implement Kubernetes API behavior.
- cloud-controller-manager (optional): Integrates with underlying cloud provider(s).

### Node Components
- kubelet: Ensures that Pods are running, including their containers.
- kube-proxy (optional): Maintains network rules on nodes to implement Services.
- Container runtime: Software responsible for running containers. Read Container Runtimes to learn more.

## Addons
- DNS: For cluster-wide DNS resolution
- Web UI (Dashboard): For cluster management via a web interface
- Container Resource Monitoring: For collecting and storing container metrics
- Cluster-level Logging: For saving container logs to a central log store

## Flexibility in Architecture