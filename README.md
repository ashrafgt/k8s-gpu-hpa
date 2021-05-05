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


#### 2. Install `dcgm-exporter`
We opted not to use the `dcgm-exporter` chart and apply the `DaemonSet` and `Service` directly.
```bash
kubectl apply -f dcgm-exporter.yaml
```
> **Note**: In `dcgm-exporter.yaml`, make sure the `hostPath` value in the `libnvidia` volume matches the `LD_LIBRARY_PATH` in your node(s). It may not necessarily be `/home/kubernetes/bin/nvidia/lib64/`.


#### 3. Test the DCGM exporter by querying it directly
Once `dcgm-exporter` is running, we can query its `/metrics` endpoint for GPU temperatures of one of the nodes for example:
```bash
kubectl port-forward svc/dcgm-exporter 9400:9400 # run this in a separate terminal
curl localhost:9400/metrics | grep dcgm_gpu_temp
```


#### 4. Add helm repositories
We add the `prometheus-community` helm repository which contains `kube-prometheus-stack` and `prometheus-adapter`:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```


#### 5. Install `kube-prometheus-stack` and compute the average GPU utilization for the deployment
As we install prometheus through the helm chart, we create two important objects:
1. An `additionalScrapeConfigs` job to scrap metrics exported by `dcgm-exporter` to `/metrics`
2. An `additionalPrometheusRules` recording rule to compute the custom metric (which we'll call `cuda_test_gpu_avg` in this example) that we'd like to use to autoscale our deployment (called `cuda-test`) 

    This metric is computed using a PromQL query where we average the positive `dcgm_gpu_utilization` values of all GPUs residing in nodes that host a replica of our deployment:
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

This is the installation command:
```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -f kube-prometheus-stack-values.yaml
```


#### 6. Test the custom metric by querying prometheus
Once prometheus is fully running, we can create our deployment (or there wouldn't be any positive `dcgm_gpu_utilization` values to average). Our test workload is a loop of calls to the `vectorAdd` script often used to test that the cluster can successfully run CUDA containers.

The custom metric should be available to query through Prometheus within 30 seconds:
```bash
kubectl apply -f cuda-test-deployment.yaml
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 # run this in a separate terminal
curl localhost:9090/api/v1/query?query=cuda_test_gpu_avg
```


#### 7. Install `prometheus-adapter`
As we install the adapter, we only need to point it to the prometheus service:
```bash
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter --set prometheus.url="http://kube-prometheus-stack-prometheus.default.svc.cluster.local"
```


#### 8. Test the custom metric export
The custom metric should available in the `custom.metrics.k8s.io` API. We use `jq` to format the response: 
```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq -r . | grep cuda_test_gpu_avg
```


#### 9. Create the deployment and horizontal pod autoscaler
We can now create our `HorizontalPodAutoscaler` resource:
```bash
kubectl apply -f cuda-test-hpa.yaml
```


#### 10. Test the horizontal pod autoscaler effects
The `HorizontalPodAutoscaler` should add an extra replica or more once the GPU utilization by the original replica reaches the target value of 4%. If the GPU utilization constantly remains under that, you can `kubectl exec` into the pod and manually double the workload to increase the value of `cuda_test_gpu_avg` by running:
```bash
for (( c=1; c<=5000; c++ )); do ./vectorAdd; done
```

Now we list the pods that belong to our deployment, and hopefully, a second replica will be added:
```bash
kubectl get pod | grep cuda-test
```
Naturally, if the usage drops low enough, a scaledown will occur.

> **Note**: The autoscaler might create more than one replica (perhaps creating as many as specified in the `maxReplicas` valie) because by default, the `dcgm-exporter` updates its metrics every 30 seconds, which means that even if the actual GPU utilization average drops down lower than the `targetValue` when a certain number of replicas is added, the value of `cuda_test_gpu_avg` may still be above the limit, causing the HPA to keep adding replicas. This default value of 30 seconds can be configured for `dcgm-exporter` through container arguments.




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
