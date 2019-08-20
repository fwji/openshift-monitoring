# openshift-monitoring
Setup Standalone Prometheus and Grafana on Openshift. The deployment yaml file are based on openshift provided prometheus and grafana templates. 

[https://github.com/openshift/origin/tree/master/examples/prometheus](https://github.com/openshift/origin/tree/master/examples/prometheus)
[https://github.com/openshift/origin/tree/master/examples/grafana](https://github.com/openshift/origin/tree/master/examples/grafana)

## Introduction
When it comes to applicaiton monitoring on openshift, Openshift Prometheus Cluster Monitoring feature is usually the first showing up on the google search result. It is a openshift feature that can be installed by one command via Openshift ansible playbooks. It packages both Prometheus Operator and Grafana in a single installation. It shoulds like a one click approach for all your monitoring needs, except well, it is not! The Prometheus Cluster Monitoring is not meant ot be modifed in anyways by the user when it comes to the scrape targets and other prometheus configuration. Its purpose is to monitor a selective of openshift and kubernetes namespaces and provide a dashboard view on Grafana to show the user how the kubernetes system is doing. As a result, you cannot really rely this feature for your day-to-day application monitoring needs. For application monitoring on Kubernetes, there are currently two approach to setup Prometheus on your cluster: 1. standalone Prometheus deployment yaml provide by Openshift and 2. Prometheus Operator. We'll introduce both approach in this guides.


## Deploy A Sample Application with MP Metrics Endpoint
1. Create a new namespaces 
```
[root@rhel7-openshift ~]# oc new-project ltf
```

2. Create a new application with metric endpoint
```
[root@rhel7-openshift ~]# oc new-app "frankji/liberty-ltf" --name ltf
```

3. Verify metrics end point http://approute.nip.io/metrics is up.


## Deploy Prometheus - Standalone deployments

1. Create a new project called prometheus
```
[root@rhel7-openshift ~]# oc new-project prometheus
```

2. Deploy Standalone Prometheus with a serviceaccount that can scrape the entire cluster
```
[root@rhel7-openshift ~]# oc new-app -f https://raw.githubusercontent.com/openshift/origin/master/examples/prometheus/prometheus.yaml -p NAMESPACE=prometheus
```

3. Edit the "prometheus" ConfigMap. Remove all exiting jobs and add the following job. 
```
...
scrape_configs:
...
      # Scrape config for API servers.
      #
      # Kubernetes exposes API servers as endpoints to the default/kubernetes
      # service so this uses `endpoints` role and uses relabelling to only keep
      # the endpoints associated with the default/kubernetes service using the
      # default named port `https`. This works for single API server deployments as
      # well as HA API server deployments.
      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
...
```

4. Kill the exsiting prometheus pod or reload the prometheus service gracefully using the command below
```
[root@rhel7-openshift ~]# oc exec prometheus-0 -c prometheus -- curl -X POST http://localhost:9090/-/reload
```

5. Edit the target application deployment yaml file to include the following annotation in the template, and restart the pod.
```
    prometheus.io/path: /metrics
    prometheus.io/port: '9080'
    prometheus.io/scrape: 'true'
```

6. Verify the scrape target is up and available in Prometheus by visitng Promethues Console -> Status -> Targets 

7. If the service endpoint is discovered, but Prometheus is reporting a *DOWN* status, you need to make Prometheus project to be globally accessible.
```
oc adm pod-network make-projects-global prometheus
```

## Deploy Prometheus - Prometheus Operator

The Prometheus Operator is an opensource project from CoreOS. It's starting to become the de facto standard for Prometheus deployments on Kubernetes system. When Prometheus Operator is installed on the Kubernetes system , user no longer needs to deal with prometheus.yml file to define the scrape targets. Instead, ServiceMonitors objects are being created for each of the service endpoint that needs to be monitored, which makes maintaning the Prometheus stack a lot easier in daily operation. An overview architecure of the prometheus is shown below:

![Prometheus Operator](https://miro.medium.com/max/1400/1*R7cnpxuu-vWYkq7ciAPA_w.png "Prometheus Operator Architecture")

There are two ways to install Prometheus Operator. One is through Openshift Operator Lifecycle Manager, which is still in its technology preview phase in Openshift 3.11. This approach will install an older version of Prometheus Operator that's supported by Openshift and Redhat. Another approach is to install Prometheus Operator using the bundle.yaml file from its official git repository at [Here](https://github.com/coreos/prometheus-operator). 

### Prometheus Operator Installation via OLM

* Make sure that you have a redhat customer portal user id before proceeding to the following sections

#### Install Prometheus Operator

1. OLM should be installed on your cluster if it has not been setup. Follow [this guide](https://docs.openshift.com/container-platform/3.11/install_config/installing-operator-framework.html) on openshift to install OLM. 

2. Create a new project called prometheus
```
[root@rhel7-openshift ~]# oc new-project prometheus
```

3. Go to Cluster Console page of the openshift web portal

4. Clic on the Operators drop-down arrow and select Catalog Sources link

5. Scroll down to the bottom and click on the *Create Subscription* button of the Prometheus Operator

6. Change the namespace to *prometheus* and click *Create*

#### Deploy a Prometheus Server

1. Click the *Cluster Service Version* on the left panel and click on the Prometheus Operator instance we just created

2. Select Create New Promtheus from the drop down button. 

3. Choose a name for your prometheus server and change the namespace to "promtheus"

4. Click on the *Create* button

5. Make sure the promtheus server pods are up and running
```
[root@rhel-2EFK ~]# oc get pods -n prometheus
NAME                                   READY     STATUS    RESTARTS   AGE
prometheus-operator-7fccbd7c74-5cz24   1/1       Running   0          3d
prometheus-server-0                    3/3       Running   1          3d
prometheus-server-1                    3/3       Running   1          3d
```
6. Expose promethues console so that it can be accessed externally.
```
oc expose svc/prometheus-operated -n prometheus
```

#### Create a ServiceMonitor to scrape the metrics endpoint

1. Make sure the target application's metrics endpoint is up and running.

2. Click the *Cluster Service Version* on the left panel and click on the Prometheus Operator instance we just created

3. Select *Create New -> Service Monitor*

4. Edit the Service Monitor yaml to include target endpoint information. You can use namespaceSelector or selector or both to filter the project or the service endpoint that needs to be scraped by Prometheus server.
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: myapp-monitor
  name: myapp-monitor
  namespace: monitoring
spec:
  endpoints:
    - interval: 30s
      path: /metrics
      port: 9080-tcp
  namespaceSelector:
    matchNames:
      - myapp
  selector:
    matchLabels:
      app: myapp
```
5. Click on the *Create* button

6. Lastly, we need to give our prometheus service view permission all the cluster namespaces
```
oc adm policy add-cluster-role-to-user view system:serviceaccount:monitoring:prometheus-k8s
```

7. Verify the scrape target is up and available in Prometheus by visitng Promethues Console -> Status -> Targets 

8. If the service endpoint is discovered, but Prometheus is reporting a *DOWN* status, you need to make Prometheus project to be globally accessible.
```
oc adm pod-network make-projects-global prometheus
```


## Deploy Grafana

1. Create a new project called grafana
```
[root@rhel-2EFK ~]# oc new-project grafana
```

2. Deploy Grafana
```
[root@rhel-2EFK ~]# oc new-app -f grafana_byo.yaml -p NAMESPACE=grafana
```

3. Grant grafana service account view access to prometheus
```
oc policy add-role-to-user view system:serviceaccount:grafana:grafana-ocp -n prometheus
```

4. In order for grafana to connect to prometheus datasource in openshift, one would need to define the datasource in a ConfigMap under grafana namespace.
  - Create a Configmap called 'grafana-datasources'
  - For the key value pair, enter 'datasources.yaml' for key
  - Enter the following for value
```
apiVersion: 1
`datasources:
  - name: "OCP Prometheus"
    type: prometheus
    access: proxy
    url: <prometheus route>
    basicAuth: false
    withCredentials: false
    isDefault: true
    jsonData:
        tlsSkipVerify: true
        "httpHeaderName1": "Authorization"
    secureJsonData:
        "httpHeaderValue1": "Bearer <grafana-ocp token>:
```
   - The <grafana-ocp token> can be acquired by the following command
  ```
  oc sa get-token grafana-ocp
  ```
  
5. Add the config map the application grafana-ocp and mount to '/usr/share/grafana/datasources'

6. Save and test the data source. You should see 'Datasource is working'
  
