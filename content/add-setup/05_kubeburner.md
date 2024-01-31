# Using kube-burner on OpenShift

# Introduction

kube-burner is an open-source command-line tool designed for the creation and destruction of Kubernetes clusters and resources at scale. It simplifies the management and deployment of complex Kubernetes environments, making it an essential tool for DevOps engineers, SysAdmins, and those involved in continuous integration and delivery pipelines. In this tutorial, we'll learn how to use kube-burner to deploy a simple benchmark application on Red Hat OpenShift.

## Prerequisites

Before you begin, make sure you have the following:

1.  A running instance of an OpenShift cluster.
2.  Kubernetes CLI (kubectl) installed and configured to communicate with your OpenShift cluster.
3.  Go programming language environment installed on your machine.
4.  Access to the GitHub repository for kube-burner: https://github.com/kube-burner/kube-burner.

## Step 1: Install kubeburner

To begin, we need to download and install the kube-burner CLI. First, let's clone the repository from GitHub:

```shell
git clone https://github.com/kube-burner/kube-burner.git
cd kube-burner
```

Now, build and install the binary for your operating system using the provided Makefile or by downloading a precompiled release:

### Using Makefile (for Linux)

```shell
make
sudo make install
```

### Or downloading a precompiled release

```shell
wget https://github.com/kube-burner/kube-burner/releases/download/v<version>/kube-burner-linux-amd64
chmod +x kube-burner-linux-amd64
sudo mv kube-burner-linux-amd64 /usr/local/bin/kube-burner
```

Replace <version> with the version number you'd like to download.

Step 2: Writing Templates for the Benchmark Application

Now, we will create YAML templates for our benchmark application which includes a web server, a curl pod used to test it, and two network policies: deny-by-default and allow-{{.Replica}}-{{.Iteration}}.

First, let's create the web server and its corresponding service using the given YAML templates below:

## Create a new directory for your project

```shell
mkdir benchmark && cd benchmark
```

```shell
cat > ubi9-template.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ubi9-{{.Replica}}-{{.Iteration}}
  labels:
    app: ubi9
    replica: {{.Replica}}
    iteration: {{.Iteration}}
spec:
  template:
    path: templates/ubi9
    values:
      replica: {{.Replica}}
      iteration: {{.Iteration}}
  restartPolicy: Always
EOF
```

```shell
cat > ubi9-service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: ubi9-{{.Replica}}-{{.Iteration}}
spec:
  selector:
    matchLabels:
      app: ubi9
      replica: {{.Replica}}
      iteration: {{.Iteration}}
  ports:
    - protocol: TCP
      name: 8000
      port: 8000
EOF
```

Next, create templates for the curl pod and network policies using the given YAML templates below:

```shell
cat > curl-template.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-{{.Replica}}-{{.Iteration}}
  labels:
    app: curl
    replica: {{.Replica}}
    iteration: {{.Iteration}}
spec:
  template:
    path: templates/curl
    values:
      replica: {{.Replica}}
      iteration: {{.Iteration}}
  restartPolicy: Always
EOF
```

```
cat > deny-by-default-template.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-by-default
spec:
  podSelector: {}
    ingress: []
EOF
```

```shell
cat > allow-{{.Replica}}-{{.Iteration}}-template.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-{{.Replica}}-{{.Iteration}}
spec:
  podSelector:
    matchLabels:
      app: ubi9
      replica: {{.Replica}}
      iteration: {{.Iteration}}
  ingress:
  - ports:
    - protocol: TCP
      port: 8000
EOF
```

## Step 3: Install Elastic, Kibana and Grafafa

To be able to monitor our benchmark we gonna install some operator. Go in the Openshift Console and Install Elasticsearch (ECK) Operator and Grafana Operator.

Then write the below configuration for the Operator

```shell
cat > elastic.yaml <<EOF
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-kube-burner
  namespace: kube-burner
spec:
  auth: {}
  http:
    service:
      metadata: {}
      spec: {}
    tls:
      certificate: {}
  image: registry.access.redhat.com/elastic/elasticsearch
  monitoring:
    logs: {}
    metrics: {}
  nodeSets:
    - config:
        node.attr.attr_name: attr_value
        node.roles:
          - master
          - data
        node.store.allow_mmap: false
      count: 3
      name: default
      podTemplate:
        metadata:
          creationTimestamp: null
          labels:
            foo: bar
        spec:
          containers:
            - name: elasticsearch
              resources:
                limits:
                  cpu: '2'
                  memory: 4Gi
                requests:
                  cpu: '1'
                  memory: 4Gi
  transport:
    service:
      metadata: {}
      spec: {}
    tls:
      certificate: {}
      certificateAuthorities: {}
  updateStrategy:
    changeBudget: {}
  version: 8.11.0
EOF
```

```shell
cat > kibana.yaml <<EOF
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-kube-burner
  namespace: kube-burner
spec:
  count: 1
  elasticsearchRef:
    name: elasticsearch-sample
  enterpriseSearchRef: {}
  http:
    service:
      metadata: {}
      spec: {}
    tls:
      certificate: {}
  image: registry.access.redhat.com/elastic/kibana
  monitoring:
    logs: {}
    metrics: {}
  podTemplate:
    metadata:
      creationTimestamp: null
      labels:
        foo: bar
    spec:
      containers:
        - name: kibana
          resources:
            limits:
              cpu: '2'
              memory: 2Gi
            requests:
              cpu: 500m
              memory: 1Gi
  version: 8.11.0
EOF
```

```shell
cat > grafana.yaml <<EOF
apiVersion: integreatly.org/v1alpha1
kind: Grafana
metadata:
  name: grafana-v8
  namespace: kube-burner
spec:
  baseImage: registry.access.redhat.com/rhel9/grafana
  config:
    auth.anonymous:
      enabled: true
    log:
      level: warn
      mode: console
    security:
      admin_password: hello
      admin_user: admin
  ingress:
    enabled: true
EOF
```

```shell
oc apply -f elastic.yaml -f kibana.yaml -f grafana.yaml
```

Then save the elasticsearch password and route with the below command.

```shell
oc get secret elasticsearch-kube-burner-es-elastic-user -n kube-burner \
   -o go-template --template="{{.data.elastic|base64decode}}"
```
```shell
oc get route -n kube-burner
```

Finally we will create an ElasticSearchDataSource that will be used to plot the content of the genereted metrics from the index.


```shell
cat > elasticGrafanaDataSource.yaml <<EOF
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDataSource
metadata:
  name: elastic-kube-burner
spec:
  datasources:
  - access: proxy
    basicAuth: true
    basicAuthPassword: YOUR_PASSWORD
    basicAuthUser: elastic
    database: kube-burner
    editable: true
    isDefault: false
    jsonData:
      tlsSkipVerify: true
      esVersion: 70
      timeField: timestamp
      timeInterval: 5s
    name: Elastic-kube-burner
    type: elasticsearch
    url: https://YOUR_ROUTE
    version: 1
  name: elastic-kube-burner.yaml
EOF
```

```shell
oc apply -f elasticGrafanaDataSource.yaml
```

## Step 4: Using kube-burner to Deploy the Benchmark Application

Now that we have our YAML templates, let's use kube-burner to create and apply them on OpenShift. Create a Kube-Burner configuration file to deploy the benchmark application using Kube-Burner. Let's start by defining the basic structure of our configuration file.

NOTE: Update the password and the route of elasticsearch with the one that you get in the previous step.

```shell
---
global:
  indexerConfig:
    esServers: [https://elastic:YOURPASSWORD@YOUR_ES_ROUTE]
    insecureSkipVerify: true
    defaultIndex: kube-burner
    type: elastic
  measurements:
    - name: podLatency
jobs:
  - name: deny-all-policy
    jobIterations: 1
    qps: 1
    burst: 1
    namespacedIterations: false
    namespace: kubelet-density-cni-networkpolicy
    jobPause: 1m
    objects:

      - objectTemplate: templates/deny-all.yml
        replicas: 1

  - name: kubelet-density-cni-networkpolicy
    jobIterations: 2
    qps: 25
    burst: 25
    namespacedIterations: false
    namespace: kubelet-density-cni-networkpolicy
    waitWhenFinished: true
    podWait: false
    preLoadImages: true
    preLoadPeriod: 2m
    objects:

      - objectTemplate: templates/ubi9-deployment.yaml
        replicas: 1

      - objectTemplate: templates/ubi9-service.yaml
        replicas: 1

      - objectTemplate: templates/allow-http.yml
        replicas: 1

      - objectTemplate: templates/curl-deployment.yaml
        replicas: 1
```

Here's an explanation of the different parts:

Global:
This section sets some general configuration parameters for the kube-burner tool. Here, we configure the ElasticSearch server endpoint, disable SSL certificate verification, set the default index name, and specify that we will use ElasticSearch as our type of metrics provider.

jobs:
The jobs section contains a list of job definitions that represent the different workloads and tests we want to run. Each job has several attributes:

  - name: A descriptive name for the job
  - jobIterations: The number of times each test is repeated (useful when testing for stability)
  - qps: The number of queries per second during a test's active phase
  - burst: The maximum number of queries per second that can be sent during a test's bursting phase
  - namespacedIterations: If true, each test will run in a separate Kubernetes namespace for better isolation
  - namespace: The namespace where the test workloads and resources will be deployed
  - jobPause: The number of seconds between job iterations (useful when testing long-running services)

objects:
The objects section lists the resource templates that kube-burner will create for each job. For example, one job might deploy a deployment, another might deploy a service, and yet another might create a network policy. The number of replicas for each template is specified in the "replicas" field.

In summary, this configuration file sets up several jobs with varying workloads (e.g., a 'deny-all-policy' job, which deploys a network policy denying all traffic) and resources (such as deployments and services), and specifies the settings for each test (like the number of iterations, QPS, and bursts). The kube-burner tool will then use this configuration file to create, deploy, and manage these resources in a Kubernetes cluster.

## Step 5: Configure metrics file

Now we will configure a metrics file that will be used to collect metrics from prometheus to collect informations about the state of the cluster during our benchmark/test.

```shell metrics.yaml
cat > elasticGrafanaDataSource.yaml <<EOF
# API server
# API server
- query: histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{apiserver="kube-apiserver", verb!~"WATCH", subresource!="log"}[2m])) by (verb,resource,subresource,instance,le)) > 0
  metricName: API99thLatency

- query: sum(irate(apiserver_request_total{apiserver="kube-apiserver",verb!="WATCH",subresource!="log"}[2m])) by (verb,instance,resource,code) > 0
  metricName: APIRequestRate

- query: sum(apiserver_current_inflight_requests{}) by (request_kind) > 0
  metricName: APIInflightRequests

# Containers & pod metrics
- query: sum(irate(container_cpu_usage_seconds_total{name!="",namespace=~"openshift-(etcd|oauth-apiserver|.*apiserver|ovn-kubernetes|sdn|ingress|authentication|.*controller-manager|.*scheduler|monitoring|logging|image-registry)"}[2m]) * 100) by (pod, namespace, node)
  metricName: podCPU

- query: sum(container_memory_rss{name!="",namespace=~"openshift-(etcd|oauth-apiserver|.*apiserver|ovn-kubernetes|sdn|ingress|authentication|.*controller-manager|.*scheduler|monitoring|logging|image-registry)"}) by (pod, namespace, node)
  metricName: podMemory

- query: (sum(rate(container_fs_writes_bytes_total{container!="",device!~".+dm.+"}[5m])) by (device, container, node) and on (node) kube_node_role{role="master"}) > 0
  metricName: containerDiskUsage

# Kubelet & CRI-O metrics
- query: sum(irate(process_cpu_seconds_total{service="kubelet",job="kubelet"}[2m]) * 100) by (node) and on (node) kube_node_role{role="worker"}
  metricName: kubeletCPU

- query: sum(process_resident_memory_bytes{service="kubelet",job="kubelet"}) by (node) and on (node) kube_node_role{role="worker"}
  metricName: kubeletMemory

- query: sum(irate(process_cpu_seconds_total{service="kubelet",job="crio"}[2m]) * 100) by (node) and on (node) kube_node_role{role="worker"}
  metricName: crioCPU

- query: sum(process_resident_memory_bytes{service="kubelet",job="crio"}) by (node) and on (node) kube_node_role{role="worker"}
  metricName: crioMemory

# Node metrics
- query: sum(irate(node_cpu_seconds_total[2m])) by (mode,instance) > 0
  metricName: nodeCPU

- query: avg(node_memory_MemAvailable_bytes) by (instance)
  metricName: nodeMemoryAvailable

- query: avg(node_memory_Active_bytes) by (instance)
  metricName: nodeMemoryActive

- query: avg(node_memory_Cached_bytes) by (instance) + avg(node_memory_Buffers_bytes) by (instance)
  metricName: nodeMemoryCached+nodeMemoryBuffers

- query: irate(node_network_receive_bytes_total{device=~"^(ens|eth|bond|team).*"}[2m])
  metricName: rxNetworkBytes

- query: irate(node_network_transmit_bytes_total{device=~"^(ens|eth|bond|team).*"}[2m])
  metricName: txNetworkBytes

- query: rate(node_disk_written_bytes_total{device!~"^(dm|rb).*"}[2m])
  metricName: nodeDiskWrittenBytes

- query: rate(node_disk_read_bytes_total{device!~"^(dm|rb).*"}[2m])
  metricName: nodeDiskReadBytes

- query: sum(rate(etcd_server_leader_changes_seen_total[2m]))
  metricName: etcdLeaderChangesRate

# Etcd metrics
- query: etcd_server_is_leader > 0
  metricName: etcdServerIsLeader

- query: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[2m]))
  metricName: 99thEtcdDiskBackendCommitDurationSeconds

- query: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[2m]))
  metricName: 99thEtcdDiskWalFsyncDurationSeconds

- query: histogram_quantile(0.99, rate(etcd_network_peer_round_trip_time_seconds_bucket[5m]))
  metricName: 99thEtcdRoundTripTimeSeconds

- query: etcd_mvcc_db_total_size_in_bytes
  metricName: etcdDBPhysicalSizeBytes

- query: etcd_mvcc_db_total_size_in_use_in_bytes
  metricName: etcdDBLogicalSizeBytes

- query: sum(rate(etcd_object_counts{}[5m])) by (resource) > 0
  metricName: etcdObjectCount

- query: sum by (cluster_version)(etcd_cluster_version)
  metricName: etcdVersion
  instant: true

# Cluster metrics
- query: sum(kube_namespace_status_phase) by (phase) > 0
  metricName: namespaceCount

- query: sum(kube_pod_status_phase{}) by (phase)
  metricName: podStatusCount

- query: count(kube_secret_info{})
  metricName: secretCount

- query: count(kube_deployment_labels{})
  metricName: deploymentCount

- query: count(kube_configmap_info{})
  metricName: configmapCount

- query: count(kube_service_info{})
  metricName: serviceCount

- query: count(openshift_route_created{})
  metricName: routeCount
  instant: true

- query: kube_node_role
  metricName: nodeRoles
  instant: true

- query: sum(kube_node_status_condition{status="true"}) by (condition)
  metricName: nodeStatus

- query: cluster_version{type="completed"}
  metricName: clusterVersion
  instant: true
EOF
```

Now launch your benchmark with the below command :

```shell
kube-burner init -m ./metrics.yaml -c ./network-policy-density.yaml -u https://$(oc get route prometheus-k8s -n openshift-monitoring -o jsonpath="{.spec.host}") --log-level=debug --token=$(oc create token prometheus-k8s -n openshift-monitoring)
```

# Validation.

TODO present the namespace kubelet-density-cni-networkpolicy.

To plot the metrics go in grafana and login with user `admin` and password `hello`

```shell
oc get route grafana-route -n kube-burner
```

Go in dashboard and click on import and upload the below json file :

```shell
https://github.com/kube-burner/kube-burner/blob/main/examples/grafana-dashboards/kube-burner-latest.json
```
