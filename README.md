# Horizontal Pod autoscaling (HPA
This repo installs and configures HPA workloads on MKE 

### Rre-reqs
1. MKE based k8s cluster 1.12.3 or above 
2. Prometheus server  

### Steps 
1. Download the helm repository for prometheus adapter 
2. Update and install the Prometheus-Adapter helm chart for custom metrics to be consumed by HPA.
3. Define and create the custom metrics so that adpater would start scapping them.
4. Create the HPA object.
5. Test scalling based on HPA

### Installation:

####1. Download helm charts
```bash
git clone https://github.com/helm/charts.git

# cd into the stable charts directory
cd charts
```
####2. Configure parameters for our installation and install the adapter. 
```
While you can add and custom tweak the installation based on the parameters, the two options that are vital are 
  a. prometheus.url
  b. port
Edit the values.yaml file to add these changes below is the sample

# Url to access prometheus
prometheus:
  url: http://prometheus.istio-system.svc.cluster.local
  port: 9090
  
# Install the adapter
helm template . --name prometheus-hpa-adapter --namespace monitoring > prometheus-adpater-install.yaml

kubectl apply -f prometheus-adpater-install.yaml

# verify deployment is successful 
kubectl get pods -n monitoring

# validate default custom metrics exposed by prometheus adapter by running command below

kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .


```


Additional configuration parameters
```
The following table lists the configurable parameters of the Prometheus Adapter chart and their default values.

| Parameter                       | Description                                                                     | Default                                     |
| ------------------------------- | ------------------------------------------------------------------------------- | --------------------------------------------|
| `affinity`                      | Node affinity                                                                   | `{}`                                        |
| `image.repository`              | Image repository                                                                | `directxman12/k8s-prometheus-adapter-amd64` |
| `image.tag`                     | Image tag                                                                       | `v0.5.0`                                    |
| `image.pullPolicy`              | Image pull policy                                                               | `IfNotPresent`                              |
| `image.pullSecrets`             | Image pull secrets                                                              | `{}`                                        |
| `logLevel`                      | Log level                                                                       | `4`                                         |
| `metricsRelistInterval`         | Interval at which to re-list the set of all available metrics from Prometheus   | `1m`                                        |
| `nodeSelector`                  | Node labels for pod assignment                                                  | `{}`                                        |
| `prometheus.url`                | Url of where we can find the Prometheus service                                 | `http://prometheus.default.svc`             |
| `prometheus.port`               | Port of where we can find the Prometheus service                                | `9090`                                      |
| `rbac.create`                   | If true, create & use RBAC resources                                            | `true`                                      |
| `resources`                     | CPU/Memory resource requests/limits                                             | `{}`                                        |
| `rules.default`                 | If `true`, enable a set of default rules in the configmap                       | `true`                                      |
| `rules.custom`                  | A list of custom configmap rules                                                | `[]`                                        |
| `rules.existing`                | The name of an existing configMap with rules. Overrides default, custom and external. | ``                                    |
| `rules.external`                | A list of custom rules for external metrics API                                 | `[]`                                        |
| `service.annotations`           | Annotations to add to the service                                               | `{}`                                        |
| `service.port`                  | Service port to expose                                                          | `443`                                       |
| `service.type`                  | Type of service to create                                                       | `ClusterIP`                                 |
| `serviceAccount.create`         | If true, create & use Serviceaccount                                            | `true`                                      |
| `serviceAccount.name`           | If not set and create is true, a name is generated using the fullname template  | ``                                          |
| `tls.enable`                    | If true, use the provided certificates. If false, generate self-signed certs    | `false`                                     |
| `tls.ca`                        | Public CA file that signed the APIService (ignored if tls.enable=false)         | ``                                          |
| `tls.key`                       | Private key of the APIService (ignored if tls.enable=false)                     | ``                                          |
| `tls.certificate`               | Public key of the APIService (ignored if tls.enable=false)                      | ``                                          |
| `tolerations`                   | List of node taints to tolerate                                                 | `[]`                                        |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,
```
#### 3. Define and create custom metric 

The prometheus adapter repository should have a configmap set up as a part of default installation. 
Make a copy and lets edit that to add our custom metrics that we desire to be scraped by the adapter, below is a sample. 
Prepare the following configMap file, replace the {{ .chart }} and {{ .release }} with the content from the default configMap file created by helm.
```
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: prometheus-adapter
    chart: {{ .chart }}
    heritage: Tiller
    release: {{ .release}}
  name: {{ .release }}-prometheus-adapter
  namespace: default
data:
  config.yaml: |
    rules:
    - seriesQuery: '{__name__= "myapp_custom_metric"}'
      seriesFilters: []
      resources:
        overrides:
          k8s_namespace:
            resource: namespace
          k8s_pod_name:
            resource: pod
      name:
        matches: "myapp_custom_metric"
        as: ""
      metricsQuery: <<.Series>>{<<.LabelMatchers>>,container_name!="POD"}
      
 ```
 validate the custom metrics 
 
 ```bash
 $ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1  | jq
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "namespaces/myapp_custom_metric",
      "singularName": "",
      "namespaced": false,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    },
    {
      "name": "pods/myapp_custom_metric",
      "singularName": "",
      "namespaced": true,
      "kind": "MetricValueList",
      "verbs": [
        "get"
      ]
    }
  ]
 ```

