# Is it Observable
<p align="center"><img src="/image/logo.png" width="40%" alt="Is It observable Logo" /></p>

## Stanza the future Log Agent of Opentelemetry
<p align="center"><img src="/image/logo_small.png" width="20%" alt="Stanza Logo" /></p>

This tutorial will be based on the popular Demo platform provided by Google : The Online Boutique
<p align="center">
<img src="image/Hipster_HeroLogoCyan.svg" width="300" alt="Online Boutique" />
</p>

## Screenshots
| Home Page                                                                                                         | Checkout Screen                                                                                                    |
| ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [![Screenshot of store homepage](image/online-boutique-frontend-1.png)](image/online-boutique-frontend-1.png) | [![Screenshot of checkout screen](image/online-boutique-frontend-2.png)](image/online-boutique-frontend-2.png) |


## Prerequisite
The following tools need to be install on your machine :
- jq
- kubectl
- git
- gcloud ( if you are using GKE)
- Helm

This tutorial will generate traces and send them to Dynatrace.
Therefore you will need a Dynatrace Tenant to be able to follow all the instructions of this tutorial .
If you don't have any dynatrace tenant , then let's start a [trial on Dynatrace](https://www.dynatrace.com/trial/)

## Deployment Steps

You will first need a Kubernetes cluster with 2 Nodes.
You can either deploy on Minikube or K3s or follow the instructions to create GKE cluster:
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
gcloud container clusters create onlineboutique \
--project=${PROJECT_ID} --zone=${ZONE} \
--machine-type=e2-standard-2 --num-nodes=4
```

### 3.Clone the Github Repository
```
git clone https://github.com/isItObservable/OpenTelemetryOperator
cd OpenTelemetryOperator
```
#### 4.Deploy Nginx Ingress Controller
```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)
kubectl apply -f nginx/deploy.yaml
```

##### 5. get the ip adress of the ingress gateway
Since we are using Ingress controller to route the traffic , we will need to get the public ip adress of our ingress.
With the public ip , we would be able to update the deployment of the ingress for :
* hipstershop
* K6
```
IP=$(kubectl get svc ngninx-nginx-ingress  -ojson | jq -j '.status.loadBalancer.ingress[].ip')
```

update the following files to update the ingress definitions :
```
sed -i "s,IP_TO_REPLACE,$IP," hipster-shop/k8s-manifest.yaml
sed -i "s,IP_TO_REPLACE,$IP," k6/k8s-manifest.yaml
```

#### 4.Prometheus
To generate traffic we are currently using K6 with the integration with Prometheus.
k6 will write load testing statistics in the the Prometheus server.
Therefore we will need to deploy the Prometheus Operator :
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack --set server.nodeSelector.node-type=observability --set prometheusOperator.nodeSelector.selector.node-type=observability  --set prometheus.nodeSelector.selector.node-type=observability --set grafana.nodeSelector.selector.node-type=observability  
```
### 5. Configure Prometheus by enabling the feature remo-writer

To measure the impact of our experiments on use traffic , we will use the load testing tool named K6.
K6 has a Prometheus integration that writes metrics to the Prometheus Server.
This integration requires to enable a feature in Prometheus named: remote-writer

To enable this feature we will need to edit the CRD containing all the settings of promethes: prometehus

To get the Prometheus object named use by prometheus we need to run the following command:
```
kubectl get Prometheus
```
here is the expected output:
```
NAME                                    VERSION   REPLICAS   AGE
prometheus-kube-prometheus-prometheus   v2.32.1   1          22h
```
We will need to add an extra property in the configuration object :
```
enableFeatures:
- remote-write-receiver
```
so to update the object :
```
kubectl edit Prometheus prometheus-kube-prometheus-prometheus
```
After the update your Prometheus object should look  like :
```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: default
  generation: 2
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 30.0.1
    chart: kube-prometheus-stack-30.0.1
    heritage: Helm
    release: prometheus
  name: prometheus-kube-prometheus-prometheus
  namespace: default
spec:
  alerting:
  alertmanagers:
  - apiVersion: v2
    name: prometheus-kube-prometheus-alertmanager
    namespace: default
    pathPrefix: /
    port: http-web
  enableAdminAPI: false
  enableFeatures:
  - remote-write-receiver
  externalUrl: http://prometheus-kube-prometheus-prometheus.default:9090
  image: quay.io/prometheus/prometheus:v2.32.1
  listenLocal: false
  logFormat: logfmt
  logLevel: info
  paused: false
  podMonitorNamespaceSelector: {}
  podMonitorSelector:
  matchLabels:
  release: prometheus
  portName: http-web
  probeNamespaceSelector: {}
  probeSelector:
  matchLabels:
  release: prometheus
  replicas: 1
  retention: 10d
  routePrefix: /
  ruleNamespaceSelector: {}
  ruleSelector:
  matchLabels:
  release: prometheus
  securityContext:
  fsGroup: 2000
  runAsGroup: 2000
  runAsNonRoot: true
  runAsUser: 1000
  serviceAccountName: prometheus-kube-prometheus-prometheus
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector:
  matchLabels:
  release: prometheus
  shards: 1
  version: v2.32.1
```

### 6. Deploy the sample applications

#### Deploy hipster-shop 
This version of the hipster-shop has the various instrumentation library added in the image of the various services.
All the generated Spans will be exported to the OpenTelemetryCollector deployed with the daemonset mode
```
kubectl create ns hipster-shop
kubectl apply -f hipster-shop/k8s-manifest.yaml -n hipster-shop
```

### 7. Deploy Stanza

#### Requirements
To be able to ingest logs  in Dynatrace, it would be requried to modify `openTelemetry-manifest_daemonset.yaml`  with
- your Dynatrace Tenant URL ( your dynatrace url would be `https://<TENANTID>.live.dynatrace.com` )
- A dynatrace API token having the right :
    * `Ingest Logs`
    * `Paas integration - Installer download`
    * `Paas integration - Support Alert`
    To generate your API token you will need to click on `Access Tokens` ( in the left menu)
    Follow the instruction described in [dynatrace's documentation](https://www.dynatrace.com/support/help/shortlink/api-authentication#generate-a-token)
    Make sure that the scope Ingest OpenTelemetry traces is enabled.
<p align="center"><img src="/image/acces_token.png" width="60%" alt="dt api scope" /></p>


#### 1. Get the cluster id of your K8s cluster
```
kubectl get namespace kube-system -o jsonpath='{.metadata.uid}'
```
#### 2. update the deployment of stanza and of the active gate

* Create a service account and cluster role for accessing the Kubernetes API.
```
kubectl apply -f dynatrace/service_account.yaml
```
* Create a secret holding the environment URL and login credentials for this registry, making sure to replace.
```
export ENVIRONMENT_URL=<with your environment URL (without 'http'). Example: environment.live.dynatrace.com>
export CLUSTERID=<YOUR CLUSTER ID>
export API_TOKEN=<YOUR API TOKEN>
export ENVIRONMENT_ID=<YOUR environementid in your environment url>
kubectl create secret docker-registry tenant-docker-registry --docker-server=${ENVIRONMENT_URL} --docker-username=${ENVIRONMENT_ID} --docker-password=${API_TOKEN} -n dynatrace
kubectl create secret generic tokens --from-literal="log-ingest=${API_TOKEN}" 
 ```

Update the file named fluentd/fluentd-manifest.yaml and activegate.yaml, by running the following command :
 ```
sed -i "s,ENVIRONMENT_ID_TO_REPLACE,$ENVIRONMENT_ID," stanza/stanza_dynatrace_config_map.yaml
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," dynatrace/activegate.yaml
sed -i "s,CLUSTER_ID_TO_REPLACE,$CLUSTERID," stanza/stanza_dynatrace_config_map.yaml
sed -i "s,API_TOKEN_TO_REPLACE,$API_TOKEN," stanza/stanza_dynatrace_config_map.yaml
sed -i "s,ENVIRONMENT_URL_TO_REPLACE,$ENVIRONMENT_URL," dynatrace/activegate.yaml
 ```

#### 10. Deploy Active gate and Stanza
```
kubectl apply -f dynatrace/activegate.yaml
kubectl apply -f stanza/stanza.yaml
kubectl create ns k6
kubectl apply -f k6/k8s-manifiest.yaml -n k6
```

#### 11. Let's have a look at the modified log stream
```
kubectl logs -l name=stanza 
```

#### 12. let's modify the configmap and use the experimental dynatrace output plugin
```
kubectl apply -f stanza/stanza_dynatrace_config_map.yaml
kubectl delete pods -l name=stanza
```