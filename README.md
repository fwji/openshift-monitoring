# openshift-monitoring
Setup Standalone Prometheus and Grafana on Openshift. The deployment yaml file are based on openshift provided prometheus and grafana templates. 

[https://github.com/openshift/origin/tree/master/examples/prometheus](https://github.com/openshift/origin/tree/master/examples/prometheus)
[https://github.com/openshift/origin/tree/master/examples/grafana](https://github.com/openshift/origin/tree/master/examples/grafana)

### Deploy A Sample Application with MP Metrics Endpoint
1. Create a new namespaces 
```
[root@rhel7-openshift ~]# oc new-project ltf
```

2. Create a new application with metric endpoint
```
[root@rhel7-openshift ~]# oc new-app "frankji/liberty-ltf" --name ltf
```

3. Verify metrics end point http://approute.nip.io/metrics is up.


### Deploy Prometheus

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

### Deploy Grafana

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
  
