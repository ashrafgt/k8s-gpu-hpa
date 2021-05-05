## Horizontal Pod Autoscaling using Nvidia GPU Metrics (WIP):

> **Please Note**: This current version is only valid for autoscaling workloads that have exclusive access to the GPUs of the nodes they're running on, which is a major limitation. Please see the limitations section at the end of the page for more details.


### Overview:
An implemention of [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) based on GPU metrics.

The approach we're experimenting with requires the following components:
- [DCGM Exporter](https://github.com/NVIDIA/gpu-monitoring-tools#dcgm-exporter) which exports GPU metrics for each GPU-capable node in the cluster that matches a certain k8s label. We selected the GPU utilization metric (`dcgm_gpu_utilization`) for this example.
- [Prometheus](https://github.com/prometheus/prometheus) which collects, transforms, and serves the GPU metrics coming from the DCGM Exporter.
- [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter) which redirects the GPU metrics served by Prometheus to the k8s custom metrics API `custom.metrics.k8s.io` so that they're used by the `HorizontalPodAutoscaler` controller to scale up our workfloads.


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
> **Note**: If you're running on GKE, EKS, or AKS and you have cluster autoscaling enabled, make sure the label is automatically attached to the node when it's created.


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
> **Note**: In `dcgm-exporter.yaml`, make sure the `hostPath` value in the `libnvidia` volume matches the `LD_LIBRARY_PATH` in your node(s). It may not necessarily be `/home/kubernetes/bin/nvidia/lib64/`.


#### 4. Test the metric export
Once the `dcgm-exporter` pod(s) start running, the metrics should be available to query through Prometheus:
```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090
curl localhost:9090/api/v1/query?query=dcgm_gpu_utilization
```


#### 5. Install `prometheus-adapter` and compute the average GPU utilization for the deployment
In `prometheus-adapter-values.yaml`, we add a `rule` of type `custom` to expose the `dcgm_gpu_utilization`.

In addition we add another `rule` of type `record` to compute the metric that will be targeted by a `HorizontalPodAutoscaler`.

This metric is computed using a PromQL query where we average the non-zero `dcgm_gpu_utilization` values of all GPUs residing in nodes that host a replica of our deployment:
```sql
avg(
    avg by(node) (dcgm_gpu_utilization > 0) 
    * on(node) group_right() 
    max by (node) (label_replace(
        (
            max by(exported_node, pod, namespace) (kube_pod_info{exported_node!=""})
            * on(pod) group_left(label_app)
            max by(pod, label_app) (kube_pod_labels{label_app="cuda-test"})
        ), 
        "node", "$1", "exported_node", "(.*)"
    ))
)
```

To install prometheus-adapter and create the two aforementionned rules:
```bash
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter --set prometheus.url="http://kube-prometheus-stack-prometheus.default.svc.cluster.local"
```


#### 6. Test the custom metric export
The custom metric should available in the `custom.metrics.k8s.io` API. We use `jq` to format the response: 
```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq -r . | grep cuda_test_gpu_avg
```


#### 7. Create the deployment and horizontal pod autoscaler
We can now create our `deployment` and `HorizontalPodAutoscaler` resources. Our test workload is a loop of calls to the `vectorAdd` script often used to test that the cluster can successfully run CUDA containers:
```bash
kubectl apply -f cuda-test-deployment.yaml # creates only a single pod
kubectl apply -f cuda-test-hpa.yaml
```


#### 8. Test the horizontal pod autoscaler effects
The `HorizontalPodAutoscaler` should add an extra replica once the GPU utilization by the original replica reaches the target value of 4%. If the GPU utilization constantly remains under that, you can exec into the pod and manually double the workload to ensure the autoscale is triggered by running:
```bash
for (( c=1; c<=5000; c++ )); do ./vectorAdd; done
```

We can then list the pods of our deployment, and hopefully, a second replica will be added:
```bash
kubectl get pod | grep cuda-test
```

If the usage drops low enough, a scaledown will occur naturally.



### Limitations:

This example has some issues that we'd like to detail:

#### 1. Target workloads must not share multi-GPU nodes with other workloads:

If you have pods from other workloads running on the same multi-GPU node as your autoscale target, the autoscale metric will mistakenly count their GPU utilization as part of your autoscale target's GPU utilization. This is because we haven't implemented a way of listing which GPU device IDs belong to a given pod. 

Consider configuring [anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity) to ensure this condition is valid.

Else, ff you're somehow able to export that information to prometheus, you can modify the autoscale metric query to solve this issue.

#### 2. A recording rule is needed for each autoscaled deployment:

To expose a custom metric to the `custom.metrics.k8s.io` API for each deployment, a recording rule that has the deployment name in the metric expression and the labels is needed. This means some duplication will occur, and if you configure the prometheus rules like this example (in the helm chart values), creating a new deployment requires redeploying the `kube-prometheus-stack` chart.

It's possible to put the rules in `configmap` and point your prometheus deployment to it, then send a reload HTTP request to the prometheus web server following the creation of your deployments:
```bash
curl -X POST http://promtheus-url:9090/-/reload
```
