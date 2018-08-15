# ambassador-istio


**Maintainers:** [feitnomore](https://github.com/feitnomore/)

This is a simple guide to implement [Ambassador](https://www.getambassador.io/) API Gateway with Tracing, Statistics and Monitoring integrated to [Istio](https://istio.io/).

By following this guide, you'll be able to see end-to-end tracing on Istio's [Jaeger](https://www.jaegertracing.io/) as well as monitoring data on [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/).

*WARNING:* Use it at your own risk.

## INSTALL ISTIO

Assuming `Kubernetes` with `RBAC` is being used, we'll use istio-demo.yaml file to install Istio on our cluster `without` mutual TLS authentication:

1. Download Istio
```sh
$ curl -L https://git.io/getLatestIstio | sh -
$ cd istio-1.0.0
$ export PATH=$PWD/bin:$PATH
```

2. Install Istio
```sh
$ kubectl apply -f install/kubernetes/helm/istio/templates/crds.yaml
$ kubectl apply -f install/kubernetes/istio-demo.yaml
```

3. Confirm Services were created
```sh
$ kubectl get services -n istio-system
```
*Note:* You should see: grafana, istio-citadel, istio-egressgateway, istio-galley, istio-ingressgateway, istio-pilot, istio-policy, istio-sidecar-injector, istio-statsd-prom-bridge, istio-telemetry, jaeger-agent, jaeger-collector, jaeger-query, prometheus, servicegraph, tracing, zipkin.

## INSTALL AMBASSADOR

Assuming `Kubernetes` with `RBAC` is being used, we'll first install `Ambassador`.

1. Create Ambassador Descriptor   
First we're going to create a YAML file called `ambassador.yaml` for our Ambassador instalation with Prometheus exporter. Please refer to Examples below for further details. The installation is being done on *default* namespace:
```yaml
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ambassador-admin
    service: ambassador-admin
  name: ambassador-admin
spec:
  type: ClusterIP
  ports:
  - name: ambassador-admin
    port: 8877
    targetPort: 8877
  selector:
    service: ambassador
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: ambassador
rules:
- apiGroups: [""]
  resources:
  - services
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["create", "update", "patch", "get", "list", "watch"]
- apiGroups: [""]
  resources:
  - secrets
  verbs: ["get", "list", "watch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ambassador
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: ambassador
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ambassador
subjects:
- kind: ServiceAccount
  name: ambassador
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: ambassador
  labels:
    app: ambassador
spec:
  replicas: 3
  template:
    metadata:
      annotations:
        sidecar.istio.io/inject: "false"
      labels:
        app: ambassador
        service: ambassador
    spec:
      serviceAccountName: ambassador
      containers:
      - name: ambassador
        image: quay.io/datawire/ambassador:0.38.0
        resources:
          limits:
            cpu: 1
            memory: 400Mi
          requests:
            cpu: 200m
            memory: 100Mi
        env:
        - name: AMBASSADOR_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        livenessProbe:
          httpGet:
            path: /ambassador/v0/check_alive
            port: 8877
          initialDelaySeconds: 30
          periodSeconds: 3
        readinessProbe:
          httpGet:
            path: /ambassador/v0/check_ready
            port: 8877
          initialDelaySeconds: 30
          periodSeconds: 3
      - name: statsd-sink
        image: datawire/prom-statsd-exporter:0.6.0
      restartPolicy: Always
```
*Note:* We're disabling Istio's sidecar injection:
```yaml
      annotations:
        sidecar.istio.io/inject: "false"
```

*Note:* We're using Prometheus Statsd Exporter:
```yaml
      - name: statsd-sink
        image: datawire/prom-statsd-exporter:0.6.0
      restartPolicy: Always
```

2. Install Ambassador  
Lets apply the YAML file created on the previous step:
```sh
$ kubectl apply -f ambassador.yaml
```

## CONFIGURE AMBASSADOR WITH TRACING

1. Determine if Istio's Zipkin is Running  
Let's figure out if Istio's Zipkin and is on `istio-system` namespace:
```sh
$ kubectl get service zipkin -n istio-system
```

2. Create Ambassador Service Descriptor  
Now we're going to create a YAML file called `ambassador-service.yaml` for our Ambassador Service configuration, using Istio's Zipkin as our `Tracing` mechanism. We're assuming the service called Zipkin is running on port 9411 on istio-system namespace. Please refer to Examples below for further details. The installation is being done on *default* namespace as well:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ambassador
  labels:
    app: ambassador
    service: ambassador-admin
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v0
      kind: TracingService
      name: tracing
      service: "zipkin.istio-system:9411"
      driver: zipkin
      config: {}
spec:
  type: LoadBalancer
  ports:
   - port: 80
     name: http
  selector:
    service: ambassador
```
*Note:* We're creating the annotation for the `TracingService` pointing to our Istio's Zipkin Service running on istio-system namespace:
```yaml
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v0
      kind: TracingService
      name: tracing
      service: "zipkin.istio-system:9411"
      driver: zipkin
      config: {}
```

3. Install Ambassador Service  
Lets apply the YAML file created on the previous step:
```sh
$ kubectl apply -f ambassador-service.yaml
```

## CONFIGURE AMBASSADOR MONITORING

1. Create Ambassador Monitor Service Descriptor  
Now we're going to create a YAML file called `ambassador-monitor.yaml` for our Ambassador Monitor Service configuration. Please refer to Examples below for further details. The installation is being done on *default* namespace as well:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ambassador-monitor
  labels:
    app: ambassador
    service: ambassador-monitor
spec:
  type: ClusterIP
  ports:
   - port: 9102
     name: prometheus-metrics
  selector:
    service: ambassador
```

2. Install Ambassador Monitor Service  
Lets apply the YAML file created on the previous step:
```sh
$ kubectl apply -f ambassador-monitor.yaml
```

## SET ISTIO PROMETHEUS CONFIGMAP

1. Edit the `Prometheus` ConfigMap either by copying it from istio-demo.yaml or using kubectl edit, and add the following scrape:
```yaml
    - job_name: 'ambassador'
      static_configs:
      - targets: ['ambassador-monitor.default:9102']
        labels:  {'application': 'ambassador'}
```

*Note:* Please refer to Examples below for further details. 

2. Apply the changes

## CONFIGURE ISTIO'S PROMETHEUS

1. Restart Istio's Prometheus:
```sh
$ export PROMETHEUS_POD=`kubectl get pods -n istio-system | grep prometheus | awk '{print $1}'`
$ kubectl delete pod $PROMETHEUS_POD -n istio-system
```

2. Make sure Prometheus is restarted:
```
$ kubectl get pods -n istio-system | grep prometheus
```

## SETUP GRAFANA DASHBOARD

1. Start Grafana Port-Forwarding
```sh
$ kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &
```

2. Open Grafana
```
http://localhost:3000/
```

3. Install Ambassador Dashboard

* Click on Create  
* Select Import  
* Enter number 4698  

4. Adjust the Dashboard Port

* Open the Imported Dashboard
* Click on Settings in the Top Right corner
* Click on Variables
* Change the port to 80 (according to the ambassador-service.yaml, if you changed the port, the change should reflect here)

5. Adjust the Dashboard Registered Services

* Open the Imported Dashboard
* Find Registered Services
* Click on the down arrow and select Edit
* Change the Metric to:
```yaml
envoy_cluster_manager_active_clusters{job="ambassador"}
```

6. Save the Changes
* Click on Save Dashboard in the Top Right corner

## REFERENCES AND IDEAS

1. [Kubernetes](https://kubernetes.io/)
2. [Ambassador](https://www.getambassador.io/)
3. [Istio](https://istio.io/)
4. [Istio Installation](https://istio.io/docs/setup/kubernetes/)
5. [Ambassador Instalation](https://www.getambassador.io/user-guide/getting-started)
6. [Ambassador Tracing](https://www.getambassador.io/reference/services/tracing-service)
7. [Ambassador Metrics](https://www.getambassador.io/reference/statistics)

## EXAMPLES

1. [ambassador.yaml](https://github.com/feitnomore/ambassador-istio/blob/master/ambassador.yaml)
2. [ambassador-service.yaml](https://github.com/feitnomore/ambassador-istio/tree/master/ambassador-service.yaml)
3. [ambassador-monitor.yaml](https://github.com/feitnomore/ambassador-istio/tree/master/ambassador-monitor.yaml)
4. [istio-prometheus-configmap.yaml](https://github.com/feitnomore/ambassador-istio/blob/master/istio-prometheus-configmap.yaml)

