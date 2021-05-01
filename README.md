## Horizontal Pod Autoscaling using Nvidia GPU Metrics (WIP):

***Please Note**: This is a Work-in-progress. The approach layed out in the following step is incomplete. We hope to have this sorted-out soon. In addition, we assume that all nodes have at most one GPU for now.*

### Overview:
An implemention of [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) based on GPU metrics.

The approach we're experimenting with requires the following components:
- [DCGM Exporter](https://github.com/NVIDIA/gpu-monitoring-tools#dcgm-exporter) which exports GPU metrics for each GPU-capable node in the cluster that matches a certain k8s label
- [Prometheus](https://github.com/prometheus/prometheus) which collects, transforms, and serves the GPU metrics coming from the DCGM Exporter
- [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter) which redirects the GPU metrics served by Prometheus to the k8s custom metrics API `custom.metrics.k8s.io` so that they're used by the `HorizontalPodAutoscaler` controller to scale up our workfloads


<br/>

### Walkthrough:

The following steps illustrate our progress:

#### 0. Setup the environment:
To follow this walkthrough, you need the following:
- A kubernetes cluster (version 1.6 or higher) with at least one Nvidia GPU (drivers properly installed and `k8s-device-plugin` deployed)
- `kubectl` and `helm` (version 3 or higher)
- `jq` and `curl` [optional]


#### 1. Label the GPU node(s)
We'll use this label when installing `dcgm-exporter` to select nodes that have Nvidia GPUs:
```bash
kubectl label nodes {node} accelerator=nvidia-gpu
```


#### 2. Add helm repositories
We add the `prometheus-community` helm repository which contains `kube-prometheus-stack` and `prometheus-adapter`:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```


#### 3. Install `kube-prometheus-stack` and `dcgm-exporter`
We opted not to use the `dcgm-exporter` chart and apply the `DaemonSet` and `Service` directly:
```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -f kube-prometheus-stack-values.yaml
kubectl apply -f dcgm-exporter.yaml
```
***Note**: In the values file `kube-prometheus-stack-values.yaml`, make sure the `hostPath` value in the `libnvidia` volume matches the `LD_LIBRARY_PATH` in your node(s). It may not necessarily be `/home/kubernetes/bin/nvidia/lib64/`.*


#### 4. Test the metric export
Once the `dcgm-exporter` pod(s) start running, the metrics should be available to query through Prometheus:
```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090
curl localhost:9090/api/v1/query?query=dcgm_gpu_utilization
```


#### 5. Install `prometheus-adapter`
In `prometheus-adapter-values.yaml`, we add a `rule` of type `custom` and another one of type `resource` to expose the `dcgm_gpu_utilization` metric for each node, then proceed to install:
```bash
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter -f prometheus-adapter-values.yaml
```


#### 6. Compute the average GPU utilization by the deployment
Currently, GPU utilization metrics are exported by the `dcgm-exporter` for each node. We need to average the `dcgm_gpu_utilization` value of all nodes that host a replica of the deployment into a `cuda_test_gpu_avg` that can be targeted by a `HorizontalPodAutoscaler`:
```bash
# TODO
```


#### 7. Test the custom metric export
The custom metric should available in the `custom.metrics.k8s.io` API. We use `jq` to format the response: 
```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq -r . | grep cuda_test_gpu_avg
```


#### 8. Create the deployment and horizontal pod autoscaler
We can now create our `deployment` and `HorizontalPodAutoscaler` resources: 
```bash
kubectl apply -f cuda-test-deployment.yaml # creates only a single pod
kubectl apply -f cuda-test-hpa.yaml
```


#### 7. Test the horizontal pod autoscaler effects
The `HorizontalPodAutoscaler` should add an extra replica once the GPU utilization by the original replica reaches the target. We can then list the pods of our deployment:
```bash
kubectl get pod | grep cuda-test
```



