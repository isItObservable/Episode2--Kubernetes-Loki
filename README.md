# Is it Observable?
<p align="center"><img src="/image/logo.png" width="40%" alt="Prometheus Logo" /></p>

## K8s and Loging with Loki
<p align="center"><img src="/image/loki_logo.png" width="40%" alt="Loki Logo" /></p>
Repository containing the files for the Episode 2 of Is it Observable : K8s and Loki


This repository showcase the usage of the Loki  by using GKE with :
- the HipsterShop


## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm
### 1.Create a Google Cloud Platform Project
```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
gcloud services enable monitoring.googleapis.com \
cloudtrace.googleapis.com \
clouddebugger.googleapis.com \
cloudprofiler.googleapis.com \
--project ${PROJECT_ID}
```
### 2.Create a GKE cluster
```
ZONE=us-central1-b
gcloud containr clusters create isitobservable \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```
### 3.Clone Github repo
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
#### Prometheus ( already done during Episde 1)
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
Deploy the ingress by making sure to replace the service name of your grafan
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
Get the ip adress of your Grafana
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
In order to build a dashboard with data stored in Loki,we first need to add a new DataSource.
In grafana, goto Configuration/Add data source.
<p align="center"><img src="/image/addsource.PNG" width="40%" alt="grafana add datasource" /></p>
Select the source Loki , and configure the url to interact with it.

Remember Grafana is hosted in the same namesapce as Loki.
So you can simply refer the loki service :
<p align="center"><img src="/image/datasource.PNG" width="40%" alt="grafana add datasource" /></p>

#### explore the data provided by Loki in Grafana 
In grafana select Explore on the main menu
Select the datasource Loki . IN the dropdow menu select the label produc -> hipster-shop
<p align="center"><img src="/image/explore.png" width="40%" alt="grafana explore" /></p>

#### Let's build a query
Loki has a specific query langage allow you to filter, transform the data and event plot a metric from your logs in a graph.
Similar to Prometheus you need to :
* filter using labels : {app="frontend",product="hipster-shop" ,stream="stdout"}
  we are here only looking at the logs from hipster-shop , app frontend and on the logs pushed in sdout.
* transform using |
 for example :
```
{job="fluent-bit",namespace="hipster-shop",stream="stdout"} | json | http_resp_took_ms >10
```
the first ```|```  specify to Grafana to use the json parser that will extract all the json properties as labels.
the second ```|``` will filter the logs on the new labels created by the json parser.
In this example we want to only get the logs where the attribute http.resp.took.ms is above 10ms ( the json parser is replace . by _)

We can then extract on field to plot it using all the various [functions available in Grafana](https://grafana.com/docs/loki/latest/logql/)

if i want to plot the response time over time i could use the function :
```
avg(avg_over_time({job="fluent-bit",namespace="hipster-shop",stream="stdout"} | json | http_resp_took_ms >10 | __error__ != "JSONParserErr"|unwrap http_resp_took_ms  [30s])) by (pod)
```