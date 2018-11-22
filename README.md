# Kubernetes autoscaling with custom metrics

In this demo we will deploy an app mockmetrics which will generate a count at `/metrics`. These metrics will be scraped by Prometheus. With the help of [`k8s-prometheus-adapter`](https://github.com/DirectXMan12/k8s-prometheus-adapter), we will create APIService `custom.metrics.k8s.io`, which then will be utilized by HPA to scale the deployment of mockmetrics app (increase number of replicas).

## Prerequisite
- You have a running Kubernetes cluster somewhere with kubectl configured to access it
- Clone the repo
  ```
  git clone https://github.com/infracloudio/kubernetes-autoscaling.git
  cd kubernetes-autoscaling
  ```
- If you are using GKE to create the cluster make sure you have created ClusterRoleBinding with 'cluster-admin' ([_instructions_](https://stackoverflow.com/a/47332612/6202405))

## Installing and configuring [helm](https://helm.sh/)
We will be using helm to install some of the components we need for this demo

- Install helm by following [these instructions](https://github.com/helm/helm/blob/master/docs/install.md)
- Create `ServiceAccount` and  `ClusterRoleBinding` for tiller
  ```
  kubectl apply -f deploy/helm/helm-tiller-rbac.yaml
  ```
  [_More information about this_](https://docs.helm.sh/using_helm/#role-based-access-control)
- Install tiller in cluster
  ```
  helm init --service-account tiller
  ```
- Verify the installation, make sure the version is 2.10+
  ```console
  $ helm version
  Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
  Server: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
  ```

## Installing prometheus-operator and Prometheus
Now, we will install [prometheus-operator](https://github.com/coreos/prometheus-operator), that will also deploy an instance of Prometheus with the help of operator

- Install prometheus-operator

  This will install `prometheus-operator` in the namespace `monitoring` and it will create [`CustomResourceDefinitions`](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions) for `AlertManager`, `Prometheus` and `ServiceMonitor` etc.
  ```
  $ helm install \
    --name mon \
    --namespace monitoring \
    stable/prometheus-operator
  ```
  ```console
  $ kubectl get crd --namespace monitoring
  NAME                                    CREATED AT
  alertmanagers.monitoring.coreos.com     2018-11-22T10:26:55Z
  prometheuses.monitoring.coreos.com      2018-11-22T10:26:55Z
  prometheusrules.monitoring.coreos.com   2018-11-22T10:26:56Z
  servicemonitors.monitoring.coreos.com   2018-11-22T10:26:56Z
  ```

- Check if all the components are deployed properly
  ```console
  $ kubectl get pods --namespace monitoring
  NAME                                                  READY     STATUS    RESTARTS   AGE
  alertmanager-mon-prometheus-operator-alertmanager-0   2/2       Running   0          6m
  mon-grafana-f7c558d65-wwbrl                           3/3       Running   0          6m
  mon-kube-state-metrics-75b445797f-7jnzg               1/1       Running   0          6m
  mon-prometheus-node-exporter-n2zmq                    1/1       Running   0          6m
  mon-prometheus-operator-operator-587ccd9566-2ddq9     1/1       Running   0          6m
  prometheus-mon-prometheus-operator-prometheus-0       3/3       Running   1          6m
  ```

## Deploying the mockmetrics application

It's a simple web server written in Golang which exposes total hit count at `/metrics` endpoint. We will create a deployment and service for it.

- This will create Deployment, Service, HorizontalPodAutoscaler in the `default` namespace and ServiceMonitor in `monitoring` namespace
  ```console
  $ kubectl create -f deploy/metrics-app/
  deployment.apps "mockmetrics-deploy" created
  horizontalpodautoscaler.autoscaling "mockmetrics-app-hpa" created
  servicemonitor.monitoring.coreos.com "mockmetrics-sm" created
  service "mockmetrics-service" created

  $ kubectl get svc,hpa
  NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
  mockmetrics-service   ClusterIP   10.39.241.189   <none>        80/TCP    2m

  NAME                  REFERENCE                       TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   <unknown>/100   1         10        1          2m
  ```
  _The `<unknown>` field will have a value once we deploy the custom metrics API server._

  The ServiceMonitor will be picked up by Prometheus, so it tells Prometheus to scrape the metrics at `/metrics` from the mockmetrics app at every 5s. Note that the ServiceMonitor is created in the namespace same as Prometheus.
  <details>

  <summary>deploy/metrics-app/mockmetrics-service-monitor.yaml</summary>

  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: mockmetrics-sm
    namespace: monitoring
    labels:
      release: mon
  spec:
    jobLabel: mockmetrics
    selector:
      matchLabels:
        app: mockmetrics-app
    namespaceSelector:
      matchNames:
      - default
    endpoints:
    - port: metrics-svc-port
      interval: 10s
      path: /metrics
  ```

  </details>

  Let's take a look at [`Horizontal Pod Autoscaler`](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
  <details>

  <summary>deploy/metrics-app/mockmetrics-hpa.yaml</summary>

  ```yaml
  apiVersion: autoscaling/v2beta1
  kind: HorizontalPodAutoscaler
  metadata:
    name: mockmetrics-app-hpa
  spec:
    scaleTargetRef:
      apiVersion: apps/v1beta1
      kind: Deployment
      name: mockmetrics-deploy
    minReplicas: 1
    maxReplicas: 10
    metrics:
    - type: Object
      object:
        target:
          kind: Service
          name: mockmetrics-service
        metricName: total_hit_count
        targetValue: 100
  ```

  </details>

  This will increase the number of replicas of `mockmetrics-deploy` when the metric `total_hit_count` associated with the service `mockmetrics-service` crosses the targetValue 1000.

- Check if the `mockmetrics-service` appears as target in the Prometheus dashboard
  ```console
  $ kubectl port-forward svc/mon-prometheus-operator-prometheus 9090:9090 --namespace monitoring
  Forwarding from 127.0.0.1:9090 -> 9090
  Forwarding from [::1]:9090 -> 9090
  ```
  Head over to http://localhost:9090/targets  
  It should look like something similar to this,
  
  ![prometheus-dashboard-targets](images/prometheus-dashboard-targets.png)

## Deploying the custom metrics API server

- Create the resources required for deploying custom metrics API server using the adapter
  ```console
  $ kubectl create -f deploy/custom-metrics-server/
  namespace "custom-metrics" created
  configmap "adapter-config" created
  serviceaccount "custom-metrics-apiserver" created
  clusterrolebinding.rbac.authorization.k8s.io "custom-metrics:system:auth-delegator" created
  rolebinding.rbac.authorization.k8s.io "custom-metrics-auth-reader" created
  clusterrolebinding.rbac.authorization.k8s.io "custom-metrics-resource-reader" created
  clusterrole.rbac.authorization.k8s.io "custom-metrics-server-resources" created
  clusterrole.rbac.authorization.k8s.io "custom-metrics-resource-reader" created
  clusterrolebinding.rbac.authorization.k8s.io "hpa-controller-custom-metrics" created
  deployment.apps "custom-metrics-apiserver" created
  service "api" created
  apiservice.apiregistration.k8s.io "v1beta1.custom.metrics.k8s.io" created
  ```
  This will create all the resources in the `custom-metrics` namespace
  - `custom-metrics-server/custom-metrics-server-config.yaml`  
    This file contains the `configMap` used to create the configuration file for the adapter, which configures how the metrics are fetched from Prometheus and how to associate those with the Kubernetes resources. More details about writing the configuration can be found [here](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config.md). [_A walkthrough of the configuration._](https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config-walkthrough.md)
  - `custom-metrics-server/custom-metrics-server-rbac.yaml`  
    Contains ServiceAccount, ClusterRoles, RoleBindings, ClusterRoleBindings to grant required permissions to the adapter
  - `custom-metrics-server/custom-metrics-server.yaml`  
    This contains definitions for Deployment of the adapter and a Service to expose it. It also contains definition of APIService, which is part of aggregation layer. It creates the API `v1beta1.custom.metrics.k8s.io`.
- Check if everything is running as expected
  ```console
  $ kubectl get svc --namespace custom-metrics
  NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
  api       ClusterIP   10.39.243.202   <none>        443/TCP   5h

  $ kubectl get apiservice | grep v1beta1.custom.metrics.k8s.io
  v1beta1.custom.metrics.k8s.io          5h
  ```
- Check if the metrics are getting collected, by querying the API `custom.metrics.k8s.io/v1beta1`
  ```console
  $ kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/*/total_hit_count"
  {"kind":"MetricValueList","apiVersion":"custom.metrics.k8s.io/v1beta1","metadata":{"selfLink":"/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/%2A/total_hit_count"},"items":[{"describedObject":{"kind":"Service","namespace":"default","name":"mockmetrics-service","apiVersion":"/__internal"},"metricName":"total_hit_count","timestamp":"2018-08-01T11:30:39Z","value":"0"}]}
  ```

## Scaling the application

- Check the `mockmetrics-app-hpa`
  ```console
  $ kubectl get hpa
  NAME                  REFERENCE                       TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   0/100     1         10        1          2h
  ```
- The mockmetrics application has following endpoints
  - `/scale/up`: keeps on increasing the `total_hit_count` when `/metrics` is accessed
  - `/scale/down`: starts decreasing the value
  - `/scale/stop`: stops the increasing or decreasing value

- Open a new terminal tab
  ```console
  $ kubectl port-forward svc/mockmetrics-service 8080:80 &
  Forwarding from 127.0.0.1:8080 -> 8080
  Forwarding from [::1]:8080 -> 8080

  $ curl localhost:8080/scale/
  stop
  ```
  Let's set the application to increase the counter
  ```console
  $ curl localhost:8080/scale/up
  Going up!
  ```
  As Prometheus is configured to scrape the metrics every 10s, the value of the `total_hit_count` will keep changing.
- Now in different terminal tab let's watch the HPA
  ```console
  $ kubectl get hpa -w
  NAME                  REFERENCE                       TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   0/100     1         10        1          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   2/100     1         10        1          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   20/100    1         10        1          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   56/100    1         10        1          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   110/100   1         10        1          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   90/100    1         10        2          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   126/100   1         10        2          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   162/100   1         10        2          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   270/100   1         10        2          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   306/100   1         10        2          11h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   171/100   1         10        4          11h
  ...
  ```
  Once the value is greater than the target, HPA will automatically increase the number of replicas for the `mockmetrics-deploy`
- To bring the value down, execute following command in the first terminal tab
  ```console
  $ curl localhost:8080/scale/down
  Going down :P

  $ kubectl get hpa -w
  NAME                  REFERENCE                       TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
  ...
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   9/100     1         10        8          12h
  mockmetrics-app-hpa   Deployment/mockmetrics-deploy   0/100     1         10        8          12h
  ```

## Other references and credits
- Writing ServiceMonitor
  - [Cluster Monitoring using Prometheus Operator](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/cluster-monitoring.md)
  - [Running Exporters](https://github.com/coreos/prometheus-operator/blob/master/Documentation/user-guides/running-exporters.md)
- [kube-prometheus](https://github.com/coreos/prometheus-operator/tree/master/helm/kube-prometheus)
- [Get Kubernetes Cluster Metrics with Prometheus in 5 Minutes](https://akomljen.com/get-kubernetes-cluster-metrics-with-prometheus-in-5-minutes/)
- [Kubernetes Metrics Server](https://github.com/kubernetes-incubator/metrics-server)
- [Custom Metrics Adapter Server Boilerplate](https://github.com/kubernetes-incubator/custom-metrics-apiserver)
- [Custom Metrics API design document](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/custom-metrics-api.md)
- [DirectXMan12/k8s-prometheus-adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter)
- [luxas/kubeadm-workshop](https://github.com/luxas/kubeadm-workshop)
- [Querying Prometheus](https://prometheus.io/docs/prometheus/latest/querying/basics/)

## Licensing
This repository is licensed under Apache License Version 2.0. See [LICENSE](./LICENSE) for the full license text.
