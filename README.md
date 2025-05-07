# How to collect logs in k8s with Loki and Promtail


This repository is here to guide you through the GitHub tutorial that goes hand-in-hand with a video available on YouTube and a detailed blog post on my website. 
Together, these resources are designed to give you a complete understanding of the topic.


Here are the links to the related assets:
- YouTube Video: [How to collect logs in k8s with Loki and Promtail](https://www.youtube.com/watch?v=XHexyDqa_S0)
- Blog Post: [How to collect logs in Kubernetes with Loki and Promtail](isitobservable.io/observability/kubernetes/how-to-collect-logs-in-kubernetes-with-loki-and-promtail)


Feel free to explore the materials, star the repository, and follow along at your own pace.


## K8s and Logging with Loki
<p align="center"><img src="/image/loki_logo.png" width="40%" alt="Loki Logo" /></p>

This repository showcases the usage of Loki by using GKE with the HipsterShop


## Prerequisite
The following tools need to be installed on your machine :
- jq
- kubectl
- git
- gcloud (if you're using GKE)
- Helm
  
### 1. Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2. Create a GKE cluster
```
ZONE=us-central1-b
gcloud containr clusters create isitobservable \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone the Github repo
```
git clone https://github.com/isItObservable/Episode2--Kubernetes-Loki
cd Episode2--Kubernetes-Loki
```
### 4. Deploy Prometheus
#### HipsterShop
```
cd hipstershop
./setup.sh
```
#### Prometheus (as done in [Episode 1](https://github.com/isItObservable/Episode1---Kubernetes-Prometheus/) )
```
helm install prometheus stable/prometheus-operator
```
#### Expose Grafana
```
kubectl get svc
kubectl edit svc prometheus-grafana
```
change to type NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  labels:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 7.0.3
    helm.sh/chart: grafana-5.3.0
  name: prometheus-grafana
  namespace: default
  resourceVersion: "89873265"
  selfLink: /api/v1/namespaces/default/services/prometheus-grafana
spec:
  clusterIP: IPADRESSS
  externalTrafficPolicy: Cluster
  ports:
  - name: service
    nodePort: 30806
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```
Deploy the ingress by making sure to replace the service name of your Grafana
```
cd ..\grafana
kubectl apply -f ingress.yaml
```
Get the login user and password of Grafana
* For the password :
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
* For the login user:
```
kubectl get secret --namespace default prometheus-grafana -o jsonpath="{.data.admin-user}" | base64 --decode
```
Get the IP address of your Grafana
```
kubectl get ingress grafana-ingress -ojson | jq  '.status.loadBalancer.ingress[].ip'
```
#### Install Loki with Promtail
```
helm repo add loki https://grafana.github.io/loki/charts
helm repo update
helm upgrade --install loki loki/loki-stack
```
#### Configure Grafana 
In order to build a dashboard with data stored in Loki, we first need to add a new DataSource.
In Grafana, go to Configuration/Add data source.
<p align="center"><img src="/image/addsource.PNG" width="60%" alt="grafana add datasource" /></p>
Select the source Loki, and configure the URL to interact with it.

Remember, Grafana is hosted in the same namespace as Loki.
So you can simply refer to the Loki service :
<p align="center"><img src="/image/datasource.PNG" width="60%" alt="grafana add datasource" /></p>

#### Explore the data provided by Loki in Grafana 
In Grafana, select Explore on the main menu
Select the datasource Loki. In the drop-down menu, select the label produc -> hipster-shop
<p align="center"><img src="/image/explore.png" width="60%" alt="grafana explore" /></p>

#### Let's build a query
Loki has a specific query language that allows you to filter, transform the data, and even plot a metric from your logs in a graph.
Similar to Prometheus, you need to :
* filter using labels : {app="frontend",product="hipster-shop" ,stream="stdout"}
  We're here only looking at the logs from hipster-shop, app frontend, and on the logs pushed in stdout.
* transform using |
 for example :
```
{job="fluent-bit",namespace="hipster-shop",stream="stdout"} | json | http_resp_took_ms >10
```
The first ```|``` specifies to Grafana to use the JSON parser that will extract all the JSON properties as labels.
The second ```|``` will filter the logs on the new labels created by the JSON parser.
In this example, we want to only get the logs where the attribute http.resp.took.ms is above 10ms ( the JSON parser is replaced by _)

We can then extract on field to plot it using all the various [functions available in Grafana](https://grafana.com/docs/loki/latest/logql/)

If I want to plot the response time over time, i could use the function :
```
avg(avg_over_time({job="fluent-bit",namespace="hipster-shop",stream="stdout"} | json | http_resp_took_ms >10 | __error__ != "JSONParserErr"|unwrap http_resp_took_ms  [30s])) by (pod)
```
