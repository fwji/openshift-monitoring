# openshift-monitoring
Setup Prometheus and Grafana on Openshift

### Deploy Prometheus

1. Create a new project called prometheus
```
[root@rhel7-openshift ~]# oc new-project prometheus
```

2. Deploy Prometheus with system:auth-delegator cluster role
```
[root@rhel7-openshift ~]# oc new-app -f https://raw.githubusercontent.com/ConSol/springboot-monitoring-example/master/templates/prometheus3.7_with_clusterrole.yaml -p NAMESPACE=prometheus
```

### Deploy Grafana

1. Create a new project called grafana
```
[root@rhel-2EFK ~]# oc new-project grafana
```

2. Deploy Grafana
```
[root@rhel-2EFK ~]# oc new-app -f https://raw.githubusercontent.com/ConSol/springboot-monitoring-example/master/templates/grafana.yaml -p NAMESPACE=grafana
```

3. Grant grafana service account view access to prometheus
```
oc policy add-role-to-user view system:serviceaccount:grafana:grafana-ocp -n prometheus
```
