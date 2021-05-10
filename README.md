## Horizontal Pod Autoscaling using Nvidia GPU Metrics:

> **Please Note**: This solution has not been extensively tested. If you encounter any unexpected, don't hesistate to open an issue.


### Overview:
An implemention of [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) based on GPU metrics using the following components:
- [DCGM Exporter](https://github.com/NVIDIA/gpu-monitoring-tools#dcgm-exporter) which exports GPU metrics for each workload that uses GPUs. We selected the GPU utilization metric (`dcgm_gpu_utilization`) for this example.
- [Prometheus](https://github.com/prometheus/prometheus) which collects the metrics coming from the DCGM Exporter and transforms them into a metric that can be used for autoscaling deployments.
- [Prometheus Adapter](https://github.com/kubernetes-sigs/prometheus-adapter) which redirects the autoscale metric served by Prometheus to the k8s custom metrics API `custom.metrics.k8s.io` so that they're used by the `HorizontalPodAutoscaler` controller.


<br/>

### Walkthrough:

The following steps detail how to configure autoscaling for a GPU workload:

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
We opted not to use the `dcgm-exporter` chart and apply the `DaemonSet` and `Service` directly:
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


#### 5. Install `kube-prometheus-stack`
As we install prometheus through the helm chart an `additionalScrapeConfigs` which creates a job to scrape metrics exported by `dcgm-exporter` to `/metrics`:
```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack -f kube-prometheus-stack-values.yaml
```

#### 6. Compute the average GPU utilization for the deployment
We can now create a recording rule to periodically compute and expose the autoscale metric (called `cuda_test_gpu_avg` in this example) for our deployment (called `cuda-test`):
```bash
kubectl apply -f cuda-test-prometheusrule.yaml
```

This metric is computed using a PromQL query where we average the `dcgm_gpu_utilization` values of all GPUs used by a replica of our deployment:
```sql
avg(
    max by(node, pod, namespace) (dcgm_gpu_utilization)
    * on(pod) group_left(label_app)
    max by(pod, label_app) (kube_pod_labels{label_app="cuda-test"})
)
```


#### 7. Test the custom metric by querying prometheus
Once prometheus is fully running, we can create our deployment (or there wouldn't be any positive `dcgm_gpu_utilization` values to average). Our test workload is a loop of calls to the `vectorAdd` script often used to test that the cluster can successfully run CUDA containers.

The custom metric should be available to query through Prometheus within 30 seconds:
```bash
kubectl apply -f cuda-test-deployment.yaml
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 # run this in a separate terminal
curl localhost:9090/api/v1/query?query=cuda_test_gpu_avg
```


#### 8. Install `prometheus-adapter`
As we install the adapter, we only need to point it to the prometheus service:
```bash
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter --set prometheus.url="http://kube-prometheus-stack-prometheus.default.svc.cluster.local"
```


#### 9. Test the custom metric export
The custom metric should available in the `custom.metrics.k8s.io` API. We use `jq` to format the response: 
```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq -r . | grep cuda_test_gpu_avg
```


#### 10. Create the deployment and horizontal pod autoscaler
We can now create our `HorizontalPodAutoscaler` resource:
```bash
kubectl apply -f cuda-test-hpa.yaml
```


#### 11. Test the horizontal pod autoscaler effects
The `HorizontalPodAutoscaler` should add an extra replica or more once the GPU utilization by the original replica reaches the target value of 4%. If the GPU utilization constantly remains under that, you can `kubectl exec` into the pod and manually double the workload to increase the value of `cuda_test_gpu_avg` by running:
```bash
for (( c=1; c<=5000; c++ )); do ./vectorAdd; done
```

Now we list the pods that belong to our deployment, and hopefully, a second replica will be added:
```bash
kubectl get pod | grep cuda-test
```
Naturally, if the usage drops low enough, a scaledown will occur.

> **Note**: By default, the autoscaler might create more than one replica (perhaps creating as many as specified in the `maxReplicas` value). This is because the `dcgm-exporter` updates its metrics every 10 seconds and the pod may take a while to pull the image and start the container. This delay in starting and capturing the effect of the replicas causes the HPA to keep adding them. You can more-or-less [control this behavior](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-configurable-scaling-behavior) if you use `autoscaling/v2beta1`.