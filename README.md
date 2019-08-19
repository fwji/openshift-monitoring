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

2. Make sure the promethues project have global network access
```
oc adm pod-network make-projects-global prometheus
```

3. Deploy Standalone Prometheus with system:auth-delegator cluster role
```
[root@rhel7-openshift ~]# oc new-app -f prometheus_byo.yaml -p NAMESPACE=prometheus
```

4. Edit the "prometheus" ConfigMap and create a new scrape job to monitor the liberty application
```
...
scrape_configs:
...
      - job_name: 'ltf'
        static_configs:
        - targets: ['<approute>']
...
```

5. Reload the prometheus service gracefully
```
[root@rhel7-openshift ~]# oc exec prometheus-0 -c prometheus -- curl -X POST http://localhost:9090/-/reload
```
6. Verify the scrape target is up and available in Prometheus by visitng Promethues Console -> Status -> Targets 

## Deploy Prometheus - Prometheus Operator

The Prometheus Operator is an opensource project form CoreOS. It's starting to become the de facto standard for Prometheus deployments on Kubernetes system. When Prometheus Operator is installed on the Kubernetes system , user no longer needs to deal with prometheus.yml file to define the scrape targets. Instead, ServiceMonitors objects are being created for each of the service endpoint that needs to be monitored, which makes maintaning the Prometheus stack a lot easier in daily operation use cases. 

There are two ways to install Prometheus Operator. One is through Openshift Operator Lifecycle Manager, which is still in its technology preview phase in Openshift 3.11. This approach will install an older version of Prometheus Operator that's supported by Openshift and Redhat. Another approach is to install Prometheus Operator using the bundle.yaml file from its official git repository. 

### Prometheus Operator Installation via OLM

1. OLM should be installed on your cluster. 


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
  
