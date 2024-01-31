# Using kube-burner to Deploy a Benchmark Application on Red Hat OpenShift with Network Policies

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
{
  "__inputs": [
    {
      "name": "DS_ELASTICSEARCH",
      "label": "Elasticsearch",
      "description": "",
      "type": "datasource",
      "pluginId": "elasticsearch",
      "pluginName": "Elasticsearch"
    }
  ],
  "__requires": [
    {
      "type": "datasource",
      "id": "elasticsearch",
      "name": "Elasticsearch",
      "version": "1.0.0"
    },
    {
      "type": "grafana",
      "id": "grafana",
      "name": "Grafana",
      "version": "7.3.0"
    },
    {
      "type": "panel",
      "id": "graph",
      "name": "Graph",
      "version": ""
    },
    {
      "type": "panel",
      "id": "stat",
      "name": "Stat",
      "version": ""
    },
    {
      "type": "panel",
      "id": "table",
      "name": "Table",
      "version": ""
    }
  ],
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": "-- Grafana --",
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "gnetId": null,
  "graphTooltip": 0,
  "id": null,
  "iteration": 1617788278675,
  "links": [],
  "panels": [
    {
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {
            "align": null,
            "displayMode": "auto",
            "filterable": false
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": [
          {
            "matcher": {
              "id": "byName",
              "options": "QPS"
            },
            "properties": [
              {
                "id": "custom.width",
                "value": 228
              }
            ]
          }
        ]
      },
      "gridPos": {
        "h": 3,
        "w": 24,
        "x": 0,
        "y": 0
      },
      "id": 327,
      "options": {
        "showHeader": true,
        "sortBy": []
      },
      "pluginVersion": "7.3.0",
      "targets": [
        {
          "bucketAggs": [],
          "metrics": [
            {
              "$$hashKey": "object:409",
              "field": "select field",
              "id": "1",
              "meta": {},
              "settings": {
                "size": 500
              },
              "type": "raw_data"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"jobSummary\"",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Job Summary",
      "transformations": [
        {
          "id": "organize",
          "options": {
            "excludeByName": {
              "_id": true,
              "_index": true,
              "_type": true,
              "jobConfig.cleanup": true,
              "jobConfig.errorOnVerify": true,
              "jobConfig.jobIterationDelay": true,
              "jobConfig.jobIterations": false,
              "jobConfig.jobPause": true,
              "jobConfig.maxWaitTimeout": true,
              "jobConfig.namespace": true,
              "jobConfig.namespaced": true,
              "jobConfig.namespacedIterations": false,
              "jobConfig.objects": true,
              "jobConfig.verifyObjects": true,
              "jobConfig.waitFor": true,
              "jobConfig.waitForDeletion": true,
              "jobConfig.waitWhenFinished": true,
              "metricName": true,
              "timestamp": true,
              "uuid": true
            },
            "indexByName": {
              "_id": 1,
              "_index": 2,
              "_type": 3,
              "elapsedTime": 7,
              "jobConfig.burst": 6,
              "jobConfig.cleanup": 11,
              "jobConfig.errorOnVerify": 12,
              "jobConfig.jobIterationDelay": 13,
              "jobConfig.jobIterations": 8,
              "jobConfig.jobPause": 14,
              "jobConfig.jobType": 9,
              "jobConfig.maxWaitTimeout": 15,
              "jobConfig.name": 4,
              "jobConfig.namespace": 16,
              "jobConfig.namespaced": 17,
              "jobConfig.namespacedIterations": 18,
              "jobConfig.objects": 19,
              "jobConfig.podWait": 10,
              "jobConfig.qps": 5,
              "jobConfig.verifyObjects": 20,
              "jobConfig.waitFor": 21,
              "jobConfig.waitForDeletion": 22,
              "jobConfig.waitWhenFinished": 23,
              "metricName": 24,
              "timestamp": 0,
              "uuid": 25
            },
            "renameByName": {
              "_type": "",
              "elapsedTime": "Elapsed time",
              "jobConfig.burst": "Burst",
              "jobConfig.cleanup": "",
              "jobConfig.errorOnVerify": "errorOnVerify",
              "jobConfig.jobIterationDelay": "jobIterationDelay",
              "jobConfig.jobIterations": "Iterations",
              "jobConfig.jobPause": "jobPause",
              "jobConfig.jobType": "Job Type",
              "jobConfig.maxWaitTimeout": "maxWaitTImeout",
              "jobConfig.name": "Name",
              "jobConfig.namespace": "namespacePrefix",
              "jobConfig.namespaced": "",
              "jobConfig.namespacedIterations": "Namespaced iterations",
              "jobConfig.objects": "",
              "jobConfig.podWait": "podWait",
              "jobConfig.qps": "QPS",
              "jobConfig.verifyObjects": "",
              "timestamp": ""
            }
          }
        }
      ],
      "type": "table"
    },
    {
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 0,
        "y": 3
      },
      "id": 347,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "center",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "7.3.0",
      "targets": [
        {
          "alias": "",
          "bucketAggs": [
            {
              "$$hashKey": "object:4404",
              "fake": true,
              "field": "labels.role.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "coun",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "count"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"nodeRoles\"",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Node count",
      "type": "stat"
    },
    {
      "datasource": "$Datasource",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 8,
        "x": 4,
        "y": 3
      },
      "id": 367,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "center",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "7.3.0",
      "targets": [
        {
          "alias": "",
          "bucketAggs": [
            {
              "$$hashKey": "object:4404",
              "fake": true,
              "field": "labels.phase.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"podStatusCount\"",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Final pod status summary",
      "type": "stat"
    },
    {
      "datasource": "$Datasource",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 4,
        "w": 4,
        "x": 12,
        "y": 3
      },
      "id": 34,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "firstNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "7.3.0",
      "targets": [
        {
          "alias": "Secrets",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"secretCount\"",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "ConfigMaps",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"configmapCount\"",
          "refId": "B",
          "timeField": "timestamp"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "",
      "type": "stat"
    },
    {
      "datasource": "$Datasource",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 2,
        "w": 6,
        "x": 16,
        "y": 3
      },
      "id": 369,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "/^labels\\.version$/",
          "values": false
        },
        "textMode": "value"
      },
      "pluginVersion": "7.3.0",
      "targets": [
        {
          "alias": "",
          "bucketAggs": [],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "versi",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {
                "size": 500
              },
              "type": "raw_data"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"clusterVersion\"",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "OpenShift version",
      "type": "stat"
    },
    {
      "datasource": "$Datasource",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 2,
        "w": 2,
        "x": 22,
        "y": 3
      },
      "id": 371,
      "options": {
        "colorMode": "value",
        "graphMode": "none",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "/^labels\\.cluster_version$/",
          "values": false
        },
        "textMode": "value"
      },
      "pluginVersion": "7.3.0",
      "targets": [
        {
          "alias": "",
          "bucketAggs": [],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "versi",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {
                "size": 500
              },
              "type": "raw_data"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"etcdVersion\"",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "Etcd version",
      "type": "stat"
    },
    {
      "datasource": "$Datasource",
      "description": "",
      "fieldConfig": {
        "defaults": {
          "custom": {},
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 2,
        "w": 8,
        "x": 16,
        "y": 5
      },
      "id": 32,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "center",
        "orientation": "vertical",
        "reduceOptions": {
          "calcs": [
            "mean"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "7.3.0",
      "targets": [
        {
          "alias": "Namespaces",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"namespaceCount\"",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "Services",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"serviceCount\"",
          "refId": "B",
          "timeField": "timestamp"
        },
        {
          "alias": "Deployments",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"deploymentCount\"",
          "refId": "C",
          "timeField": "timestamp"
        }
      ],
      "timeFrom": null,
      "timeShift": null,
      "title": "",
      "type": "stat"
    },
    {
      "collapsed": true,
      "datasource": "${DS_ELASTICSEARCH}",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 7
      },
      "id": 417,
      "panels": [
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 8
          },
          "hiddenSeries": false,
          "id": 413,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": false,
            "hideEmpty": true,
            "hideZero": true,
            "max": true,
            "min": false,
            "rightSide": false,
            "show": true,
            "sideWidth": null,
            "sort": "max",
            "sortDesc": false,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "{{labels.pod.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:182",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "4",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:4367",
                  "fake": true,
                  "field": "labels.container.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:4301",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:4299",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: containerCPU-Prometheus",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Prometheus CPU usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:4320",
              "format": "percent",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:4321",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 8
          },
          "hiddenSeries": false,
          "id": 415,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "hideEmpty": true,
            "hideZero": true,
            "max": true,
            "min": false,
            "rightSide": false,
            "show": true,
            "sideWidth": null,
            "sort": "max",
            "sortDesc": false,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "{{labels.pod.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:182",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "4",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:4367",
                  "fake": true,
                  "field": "labels.container.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:4301",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:4299",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: containerMemory-Prometheus",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Prometheus RSS Memory usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:4320",
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:4321",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        }
      ],
      "title": "Monitoring stack",
      "type": "row"
    },
    {
      "collapsed": true,
      "datasource": "${DS_ELASTICSEARCH}",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 8
      },
      "id": 21,
      "panels": [
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 0,
            "y": 16
          },
          "hiddenSeries": false,
          "id": 18,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": true,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "",
              "bucketAggs": [
                {
                  "$$hashKey": "object:4404",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"99thEtcdDiskWalFsyncDurationSeconds\"",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "etcd 99th disk WAL fsync latency",
          "tooltip": {
            "shared": true,
            "sort": 2,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:65",
              "format": "s",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:66",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 12,
            "y": 16
          },
          "hiddenSeries": false,
          "id": 19,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": true,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "",
              "bucketAggs": [
                {
                  "$$hashKey": "object:4404",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"99thEtcdDiskBackendCommitDurationSeconds\"",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "etcd 99th disk backend commit latency",
          "tooltip": {
            "shared": true,
            "sort": 2,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:65",
              "format": "s",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:66",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {},
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 25
          },
          "hiddenSeries": false,
          "id": 314,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": true,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Etcd leader changes",
              "bucketAggs": [
                {
                  "$$hashKey": "object:280",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:278",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: etcdLeaderChangesRate",
              "refId": "A",
              "timeField": "timestamp"
            },
            {
              "alias": "Current leader: {{term labels.pod.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:375",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:280",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:278",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: etcdServerIsLeader",
              "refId": "B",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Etcd leader",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:301",
              "format": "none",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:302",
              "decimals": null,
              "format": "none",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 25
          },
          "hiddenSeries": false,
          "id": 325,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": false,
            "hideEmpty": true,
            "hideZero": true,
            "max": true,
            "min": false,
            "rightSide": true,
            "show": true,
            "sort": "total",
            "sortDesc": true,
            "total": true,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "{{term labels.resource.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:563",
                  "fake": true,
                  "field": "labels.resource.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:280",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:278",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: etcdObjectCount",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Etcd object count rate",
          "tooltip": {
            "shared": true,
            "sort": 2,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:301",
              "format": "none",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:302",
              "decimals": null,
              "format": "none",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {
                "align": null,
                "filterable": false
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 33
          },
          "hiddenSeries": false,
          "id": 391,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": false,
            "max": true,
            "min": false,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [
            {
              "$$hashKey": "object:396",
              "alias": "/.*Logical.*/",
              "yaxis": 2
            }
          ],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Physical size: {{labels.pod.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:322",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:259",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "auto",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "hide": false,
              "metrics": [
                {
                  "$$hashKey": "object:278",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: etcdDBPhysicalSizeBytes",
              "refId": "A",
              "timeField": "timestamp"
            },
            {
              "alias": "Logical size: {{labels.pod.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:337",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:280",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "hide": false,
              "metrics": [
                {
                  "$$hashKey": "object:278",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: etcdDBLogicalSizeBytes",
              "refId": "B",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Etcd DB size",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:301",
              "format": "decbytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:302",
              "decimals": null,
              "format": "decbytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {
                "align": null,
                "filterable": false
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 33
          },
          "hiddenSeries": false,
          "id": 393,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "hideEmpty": false,
            "hideZero": false,
            "max": true,
            "min": false,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [
            {
              "$$hashKey": "object:396",
              "alias": "/.*Logical.*/",
              "yaxis": 2
            }
          ],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "{{labels.pod.keyword}} to {{labels.To.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:806",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "4",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:322",
                  "fake": true,
                  "field": "labels.To.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:259",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "auto",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "hide": false,
              "metrics": [
                {
                  "$$hashKey": "object:278",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: 99thEtcdRoundTripTimeSeconds",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Etcd 99th network peer roundtrip time",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:301",
              "format": "s",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:302",
              "decimals": null,
              "format": "decbytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": false
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 41
          },
          "hiddenSeries": false,
          "id": 31,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": true,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "",
              "bucketAggs": [
                {
                  "$$hashKey": "object:4404",
                  "fake": true,
                  "field": "labels.condition.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"nodeStatus\"",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Node status summary",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:430",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:431",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 41
          },
          "hiddenSeries": false,
          "id": 30,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": true,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "",
              "bucketAggs": [
                {
                  "$$hashKey": "object:4404",
                  "fake": true,
                  "field": "labels.phase.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"podStatusCount\"",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Pod status summary",
          "tooltip": {
            "shared": true,
            "sort": 2,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:65",
              "format": "none",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:66",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 49
          },
          "hiddenSeries": false,
          "id": 141,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": false,
            "hideEmpty": false,
            "hideZero": false,
            "max": true,
            "min": false,
            "rightSide": true,
            "show": true,
            "sideWidth": null,
            "sort": "max",
            "sortDesc": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "bucketAggs": [
                {
                  "$$hashKey": "object:4367",
                  "fake": true,
                  "field": "labels.verb.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": 0,
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:4301",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:4299",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: API99thLatency",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "API P99 latency",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:4320",
              "format": "s",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:4321",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        }
      ],
      "title": "Cluster status",
      "type": "row"
    },
    {
      "collapsed": true,
      "datasource": "${DS_ELASTICSEARCH}",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 9
      },
      "id": 320,
      "panels": [
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 8
          },
          "hiddenSeries": false,
          "id": 316,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "max": true,
            "min": false,
            "rightSide": false,
            "show": true,
            "sort": "avg",
            "sortDesc": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "{{field}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:80",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "auto",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:78",
                  "field": "podReadyLatency",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                },
                {
                  "$$hashKey": "object:139",
                  "field": "schedulingLatency",
                  "id": "3",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                },
                {
                  "$$hashKey": "object:1778",
                  "field": "initializedLatency",
                  "id": "4",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: podLatencyMeasurement",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Average pod latency",
          "tooltip": {
            "shared": true,
            "sort": 2,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:101",
              "format": "ms",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:102",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "align": "left"
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 5000
                  }
                ]
              },
              "unit": "ms"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 8
          },
          "id": 318,
          "options": {
            "colorMode": "value",
            "graphMode": "none",
            "justifyMode": "auto",
            "orientation": "auto",
            "reduceOptions": {
              "calcs": [
                "mean"
              ],
              "fields": "",
              "values": false
            },
            "textMode": "auto"
          },
          "pluginVersion": "7.3.0",
          "targets": [
            {
              "alias": "{{field}} {{term quantileName.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:485",
                  "fake": true,
                  "field": "quantileName.keyword",
                  "id": "5",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "1",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:342",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "auto",
                    "min_doc_count": 0,
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:340",
                  "field": "P99",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "max"
                }
              ],
              "query": "uuid: $uuid AND metricName.keyword: podLatencyQuantilesMeasurement",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "timeFrom": null,
          "timeShift": null,
          "title": "Pod latencies summary",
          "type": "stat"
        }
      ],
      "title": "Pod latency stats",
      "type": "row"
    },
    {
      "collapsed": true,
      "datasource": "${DS_ELASTICSEARCH}",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 10
      },
      "id": 121,
      "panels": [
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 9
          },
          "hiddenSeries": false,
          "id": 116,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "max": true,
            "min": false,
            "rightSide": true,
            "show": true,
            "sort": "avg",
            "sortDesc": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "bucketAggs": [
                {
                  "$$hashKey": "object:134",
                  "fake": true,
                  "field": "labels.node.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "1",
                    "size": "5"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:61",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:59",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND (metricName.keyword: top3KubeletCPU OR metricName.keyword: kubeletCPU)",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Top 5 Kubelet process CPU usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:80",
              "format": "percent",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:81",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 9
          },
          "hiddenSeries": false,
          "id": 119,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": false,
            "max": true,
            "min": false,
            "rightSide": true,
            "show": true,
            "sort": "max",
            "sortDesc": false,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "bucketAggs": [
                {
                  "$$hashKey": "object:134",
                  "fake": true,
                  "field": "labels.node.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "1",
                    "size": "5"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:61",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:59",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND (metricName.keyword: top3KubeletMemory OR metricName.keyword: kubeletMemory)",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Top 5 Kubelet process RSS memory usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:80",
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:81",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 17
          },
          "hiddenSeries": false,
          "id": 117,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "max": true,
            "min": false,
            "rightSide": true,
            "show": true,
            "sort": "max",
            "sortDesc": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "bucketAggs": [
                {
                  "$$hashKey": "object:134",
                  "fake": true,
                  "field": "labels.node.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "1",
                    "size": "5"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:61",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:59",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND (metricName.keyword: top3CrioCPU OR metricName.keyword: crioCPU)",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Top 5 CRI-O process CPU usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:80",
              "format": "percent",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:81",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 17
          },
          "hiddenSeries": false,
          "id": 118,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": false,
            "max": true,
            "min": false,
            "rightSide": true,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "bucketAggs": [
                {
                  "$$hashKey": "object:134",
                  "fake": true,
                  "field": "labels.node.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "1",
                    "size": "5"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:61",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:59",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND (metricName.keyword: top3CrioMemory OR metricName.keyword: crioMemory)",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Top 5 CRI-O process RSS memory usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:80",
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:81",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        }
      ],
      "title": "Cluster Kubelet & CRI-O",
      "type": "row"
    },
    {
      "collapsed": false,
      "datasource": "${DS_ELASTICSEARCH}",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 11
      },
      "id": 11,
      "panels": [],
      "repeat": "master",
      "title": "Master: $master",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 9,
        "w": 12,
        "x": 0,
        "y": 12
      },
      "hiddenSeries": false,
      "id": 108,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": false,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "{{labels.pod.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:159",
              "fake": true,
              "field": "labels.pod.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "1",
                "size": "5"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"podCPU\" AND labels.node.keyword: $master AND labels.namespace.keyword: $namespace",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Top 5 CPU  usage $master",
      "tooltip": {
        "shared": true,
        "sort": 2,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:65",
          "format": "percent",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:66",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 9,
        "w": 12,
        "x": 12,
        "y": 12
      },
      "hiddenSeries": false,
      "id": 110,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": true,
        "min": false,
        "rightSide": false,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "{{labels.pod.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:159",
              "fake": true,
              "field": "labels.pod.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "1",
                "size": "5"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"podMemory\" AND labels.node.keyword: $master AND labels.namespace.keyword: $namespace",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Top 5 memory RSS $master",
      "tooltip": {
        "shared": true,
        "sort": 2,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:65",
          "decimals": null,
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:66",
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": false
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 21
      },
      "hiddenSeries": false,
      "id": 323,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "{{labels.container.keyword}}: {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:585",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "4",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:576",
              "fake": true,
              "field": "labels.container.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:543",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "auto",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:541",
              "field": "value",
              "id": "1",
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"containerDiskUsage\" AND labels.node.keyword: $master",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Container disk write rate: $master",
      "tooltip": {
        "shared": true,
        "sort": 2,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:616",
          "format": "Bps",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:617",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 9,
        "w": 12,
        "x": 12,
        "y": 21
      },
      "hiddenSeries": false,
      "id": 9,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": true,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "available",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"nodeMemoryAvailable\" AND labels.instance.keyword: $master",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "cached+buffers",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"nodeMemoryCached+nodeMemoryBuffers\" AND labels.instance.keyword: $master",
          "refId": "B",
          "timeField": "timestamp"
        },
        {
          "alias": "active",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"nodeMemoryActive\" AND labels.instance.keyword: $master",
          "refId": "C",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Memory $master",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:65",
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:66",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 9,
        "w": 12,
        "x": 0,
        "y": 29
      },
      "hiddenSeries": false,
      "id": 2,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "{{labels.mode.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:133",
              "fake": true,
              "field": "labels.mode.keyword",
              "id": "4",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "1",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "auto",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": "_value*100",
              "meta": {},
              "settings": {
                "script": {
                  "inline": "_value*100"
                }
              },
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"nodeCPU\" AND labels.instance.keyword: $master",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "CPU $master",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:65",
          "format": "percent",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:66",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 30
      },
      "hiddenSeries": false,
      "id": 35,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "$$hashKey": "object:1248",
          "alias": "/Read.+/",
          "transform": "negative-Y"
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "Written bytes - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:554",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1089",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:1087",
              "field": "value",
              "id": "1",
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND metricName: \"nodeDiskWrittenBytes\" AND labels.instance.keyword: \"$master\"",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "Read bytes - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:571",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1195",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:1193",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND  metricName: \"nodeDiskReadBytes\" AND labels.instance.keyword: \"$master\"",
          "refId": "B",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Disk $master",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:1121",
          "format": "Bps",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:1122",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 38
      },
      "hiddenSeries": false,
      "id": 4,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "$$hashKey": "object:1248",
          "alias": "/rxNetworkBytes.*/",
          "transform": "negative-Y"
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "txNetworkBytes - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:1174",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1089",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:1087",
              "field": "value",
              "id": "1",
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND metricName: \"txNetworkBytes\" AND labels.instance.keyword: \"$master\"",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "rxNetworkBytes - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:1215",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1195",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:1193",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND  metricName: \"rxNetworkBytes\" AND labels.instance.keyword: \"$master\"",
          "refId": "B",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Network bytes $master",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:1121",
          "format": "Bps",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:1122",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 38
      },
      "hiddenSeries": false,
      "id": 6,
      "legend": {
        "avg": false,
        "current": false,
        "hideEmpty": true,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "$$hashKey": "object:1826",
          "alias": "/rxDroppedPackets.*/",
          "transform": "negative-Y"
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "txDroppedPackets - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:1751",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1370",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "hide": false,
          "metrics": [
            {
              "$$hashKey": "object:1368",
              "field": "value",
              "id": "1",
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND metricName: \"txDroppedPackets\" AND labels.instance.keyword: $master",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "rxDroppedPackets - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:1768",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1594",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "hide": false,
          "metrics": [
            {
              "$$hashKey": "object:1592",
              "field": "value",
              "id": "1",
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND metricName: \"rxDroppedPackets\" AND labels.instance.keyword: $master",
          "refId": "B",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Dropped packets $master",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:1701",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:1702",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": false,
      "datasource": "${DS_ELASTICSEARCH}",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 116
      },
      "id": 37,
      "panels": [],
      "repeat": "worker",
      "title": "Worker: $worker",
      "type": "row"
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 9,
        "w": 12,
        "x": 0,
        "y": 117
      },
      "hiddenSeries": false,
      "id": 39,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": false,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "{{labels.pod.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:159",
              "fake": true,
              "field": "labels.pod.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "1",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"podCPU\" AND labels.node.keyword: \"$worker\" AND labels.namespace.keyword: $namespace",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Top 5 CPU usage $worker",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:65",
          "format": "percent",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:66",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 9,
        "w": 12,
        "x": 12,
        "y": 117
      },
      "hiddenSeries": false,
      "id": 85,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": true,
        "min": false,
        "rightSide": false,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "{{labels.pod.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:159",
              "fake": true,
              "field": "labels.pod.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "1",
                "size": "5"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"podMemory\" AND labels.node.keyword: \"$worker\" AND labels.namespace.keyword: $namespace",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Top 5 memory RSS $worker",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:65",
          "decimals": null,
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:66",
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 126
      },
      "hiddenSeries": false,
      "id": 68,
      "legend": {
        "alignAsTable": true,
        "avg": true,
        "current": false,
        "max": false,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "{{labels.mode.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:133",
              "fake": true,
              "field": "labels.mode.keyword",
              "id": "4",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "1",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "auto",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": "_value*100",
              "meta": {},
              "settings": {
                "script": {
                  "inline": "_value*100"
                }
              },
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"nodeCPU\" AND labels.instance.keyword: \"$worker\"",
          "refId": "A",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "CPU $worker",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:65",
          "format": "percent",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:66",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 126
      },
      "hiddenSeries": false,
      "id": 41,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "max": true,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "available",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"nodeMemoryAvailable\" AND labels.instance.keyword: \"$worker\"",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "cached+buffers",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"nodeMemoryCached+nodeMemoryBuffers\" AND labels.instance.keyword: \"$worker\"",
          "refId": "B",
          "timeField": "timestamp"
        },
        {
          "alias": "active",
          "bucketAggs": [
            {
              "$$hashKey": "object:33",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": "1",
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:31",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid.keyword: $uuid AND metricName: \"nodeMemoryActive\" AND labels.instance.keyword: \"$worker\"",
          "refId": "C",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Memory $worker",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:65",
          "format": "bytes",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:66",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 134
      },
      "hiddenSeries": false,
      "id": 43,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "$$hashKey": "object:1248",
          "alias": "/rxNetworkBytes.*/",
          "transform": "negative-Y"
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "txNetworkBytes - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:1174",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1089",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": 0,
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:1087",
              "field": "value",
              "id": "1",
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND metricName: \"txNetworkBytes\" AND labels.instance.keyword: \"$worker\"",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "rxNetworkBytes - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:1215",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1195",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": 0,
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:1193",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND metricName: \"rxNetworkBytes\" AND labels.instance.keyword: \"$worker\"",
          "refId": "B",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Network bytes $worker",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:1121",
          "format": "Bps",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:1122",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 134
      },
      "hiddenSeries": false,
      "id": 67,
      "legend": {
        "avg": false,
        "current": false,
        "max": false,
        "min": false,
        "show": true,
        "total": false,
        "values": false
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "$$hashKey": "object:1248",
          "alias": "/Read.+/",
          "transform": "negative-Y"
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "Written bytes - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:554",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1089",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": 0,
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:1087",
              "field": "value",
              "id": "1",
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND metricName: \"nodeDiskWrittenBytes\" AND labels.instance.keyword: \"$worker\"",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "Read bytes - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:571",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": "1",
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1195",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": 0,
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "metrics": [
            {
              "$$hashKey": "object:1193",
              "field": "value",
              "id": "1",
              "inlineScript": null,
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND  metricName: \"nodeDiskReadBytes\" AND labels.instance.keyword: \"$worker\"",
          "refId": "B",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Disk $worker",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:1121",
          "format": "Bps",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:1122",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "aliasColors": {},
      "bars": false,
      "dashLength": 10,
      "dashes": false,
      "datasource": "$Datasource",
      "fieldConfig": {
        "defaults": {
          "custom": {}
        },
        "overrides": []
      },
      "fill": 1,
      "fillGradient": 0,
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 142
      },
      "hiddenSeries": false,
      "id": 63,
      "legend": {
        "alignAsTable": true,
        "avg": false,
        "current": false,
        "hideEmpty": true,
        "hideZero": false,
        "max": true,
        "min": false,
        "rightSide": true,
        "show": true,
        "total": false,
        "values": true
      },
      "lines": true,
      "linewidth": 1,
      "nullPointMode": "null",
      "options": {
        "alertThreshold": true
      },
      "percentage": false,
      "pluginVersion": "7.3.0",
      "pointradius": 2,
      "points": false,
      "renderer": "flot",
      "seriesOverrides": [
        {
          "$$hashKey": "object:1826",
          "alias": "/rxDroppedPackets.*/",
          "transform": "negative-Y"
        }
      ],
      "spaceLength": 10,
      "stack": false,
      "steppedLine": false,
      "targets": [
        {
          "alias": "txDroppedPackets - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:1751",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": 0,
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1370",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": 0,
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "hide": false,
          "metrics": [
            {
              "$$hashKey": "object:1368",
              "field": "value",
              "id": "1",
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND metricName: txDroppedPackets AND labels.instance.keyword: $worker",
          "refId": "A",
          "timeField": "timestamp"
        },
        {
          "alias": "rxDroppedPackets - {{labels.device.keyword}}",
          "bucketAggs": [
            {
              "$$hashKey": "object:1768",
              "fake": true,
              "field": "labels.device.keyword",
              "id": "3",
              "settings": {
                "min_doc_count": 0,
                "order": "desc",
                "orderBy": "_term",
                "size": "10"
              },
              "type": "terms"
            },
            {
              "$$hashKey": "object:1594",
              "field": "timestamp",
              "id": "2",
              "settings": {
                "interval": "30s",
                "min_doc_count": 0,
                "trimEdges": 0
              },
              "type": "date_histogram"
            }
          ],
          "hide": false,
          "metrics": [
            {
              "$$hashKey": "object:1592",
              "field": "value",
              "id": "1",
              "meta": {},
              "settings": {},
              "type": "avg"
            }
          ],
          "query": "uuid: $uuid AND metricName: rxDroppedPackets AND labels.instance.keyword: $worker",
          "refId": "B",
          "timeField": "timestamp"
        }
      ],
      "thresholds": [],
      "timeFrom": null,
      "timeRegions": [],
      "timeShift": null,
      "title": "Dropped packets $worker",
      "tooltip": {
        "shared": true,
        "sort": 0,
        "value_type": "individual"
      },
      "type": "graph",
      "xaxis": {
        "buckets": null,
        "mode": "time",
        "name": null,
        "show": true,
        "values": []
      },
      "yaxes": [
        {
          "$$hashKey": "object:1701",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        },
        {
          "$$hashKey": "object:1702",
          "format": "short",
          "label": null,
          "logBase": 1,
          "max": null,
          "min": null,
          "show": true
        }
      ],
      "yaxis": {
        "align": false,
        "alignLevel": null
      }
    },
    {
      "collapsed": true,
      "datasource": "${DS_ELASTICSEARCH}",
      "gridPos": {
        "h": 1,
        "w": 24,
        "x": 0,
        "y": 150
      },
      "id": 51,
      "panels": [
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 0,
            "y": 10
          },
          "hiddenSeries": false,
          "id": 112,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "max": false,
            "min": false,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "scopedVars": {
            "infra": {
              "selected": true,
              "text": "ip-10-0-152-106.us-west-2.compute.internal",
              "value": "ip-10-0-152-106.us-west-2.compute.internal"
            }
          },
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "{{labels.pod.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:159",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "1",
                    "size": "5"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"podCPU\" AND labels.node.keyword: $infra AND labels.namespace.keyword: $namespace",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Top 5 CPU usage $infra",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:65",
              "format": "percent",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:66",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 12,
            "y": 10
          },
          "hiddenSeries": false,
          "id": 114,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": false,
            "max": true,
            "min": false,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "scopedVars": {
            "infra": {
              "selected": true,
              "text": "ip-10-0-152-106.us-west-2.compute.internal",
              "value": "ip-10-0-152-106.us-west-2.compute.internal"
            }
          },
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "{{labels.pod.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:159",
                  "fake": true,
                  "field": "labels.pod.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "1",
                    "size": "5"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"podMemory\" AND labels.node.keyword: $infra AND labels.namespace.keyword: $namespace",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Top 5 memory RSS $infra",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:65",
              "decimals": null,
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:66",
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 0,
            "y": 19
          },
          "hiddenSeries": false,
          "id": 57,
          "legend": {
            "alignAsTable": true,
            "avg": true,
            "current": false,
            "max": false,
            "min": false,
            "rightSide": true,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "scopedVars": {
            "infra": {
              "selected": true,
              "text": "ip-10-0-152-106.us-west-2.compute.internal",
              "value": "ip-10-0-152-106.us-west-2.compute.internal"
            }
          },
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "{{labels.mode.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:133",
                  "fake": true,
                  "field": "labels.mode.keyword",
                  "id": "4",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "auto",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": "_value*100",
                  "meta": {},
                  "settings": {
                    "script": {
                      "inline": "_value*100"
                    }
                  },
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"nodeCPU\" AND labels.instance.keyword: $infra",
              "refId": "A",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "CPU $infra",
          "tooltip": {
            "shared": true,
            "sort": 2,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:65",
              "format": "percent",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:66",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 12,
            "y": 19
          },
          "hiddenSeries": false,
          "id": 59,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": false,
            "max": true,
            "min": false,
            "rightSide": true,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "scopedVars": {
            "infra": {
              "selected": true,
              "text": "ip-10-0-152-106.us-west-2.compute.internal",
              "value": "ip-10-0-152-106.us-west-2.compute.internal"
            }
          },
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "available",
              "bucketAggs": [
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"nodeMemoryAvailable\" AND labels.instance.keyword: $infra",
              "refId": "A",
              "timeField": "timestamp"
            },
            {
              "alias": "cached+buffers",
              "bucketAggs": [
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"nodeMemoryCached+nodeMemoryBuffers\" AND labels.instance.keyword: $infra",
              "refId": "B",
              "timeField": "timestamp"
            },
            {
              "alias": "available",
              "bucketAggs": [
                {
                  "$$hashKey": "object:33",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": "1",
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:31",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid.keyword: $uuid AND metricName: \"nodeMemoryActive\" AND labels.instance.keyword: $infra",
              "refId": "C",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Memory $infra",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:65",
              "format": "bytes",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:66",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 28
          },
          "hiddenSeries": false,
          "id": 53,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": true,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "scopedVars": {
            "infra": {
              "selected": true,
              "text": "ip-10-0-152-106.us-west-2.compute.internal",
              "value": "ip-10-0-152-106.us-west-2.compute.internal"
            }
          },
          "seriesOverrides": [
            {
              "$$hashKey": "object:1248",
              "alias": "/rxNetworkBytes.*/",
              "transform": "negative-Y"
            }
          ],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "txNetworkBytes - {{labels.device.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:1174",
                  "fake": true,
                  "field": "labels.device.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:1089",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": 0,
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:1087",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName: \"txNetworkBytes\" AND labels.instance.keyword: \"$infra\"",
              "refId": "A",
              "timeField": "timestamp"
            },
            {
              "alias": "rxNetworkBytes - {{labels.device.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:1215",
                  "fake": true,
                  "field": "labels.device.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:1195",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": 0,
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:1193",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND  metricName: \"rxNetworkBytes\" AND labels.instance.keyword: \"$infra\"",
              "refId": "B",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Network bytes $infra",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:1121",
              "format": "Bps",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:1122",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 28
          },
          "hiddenSeries": false,
          "id": 55,
          "legend": {
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "show": true,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "scopedVars": {
            "infra": {
              "selected": true,
              "text": "ip-10-0-152-106.us-west-2.compute.internal",
              "value": "ip-10-0-152-106.us-west-2.compute.internal"
            }
          },
          "seriesOverrides": [
            {
              "$$hashKey": "object:1248",
              "alias": "/Read.+/",
              "transform": "negative-Y"
            }
          ],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "Written bytes - {{labels.device.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:554",
                  "fake": true,
                  "field": "labels.device.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:1089",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": 0,
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:1087",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName: \"nodeDiskWrittenBytes\" AND labels.instance.keyword: \"$infra\"",
              "refId": "A",
              "timeField": "timestamp"
            },
            {
              "alias": "Read bytes - {{labels.device.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:571",
                  "fake": true,
                  "field": "labels.device.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": "1",
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:1195",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": 0,
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "metrics": [
                {
                  "$$hashKey": "object:1193",
                  "field": "value",
                  "id": "1",
                  "inlineScript": null,
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND  metricName: \"nodeDiskReadBytes\" AND labels.instance.keyword: \"$infra\"",
              "refId": "B",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Disk $infra",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:1121",
              "format": "Bps",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:1122",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        },
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "$Datasource",
          "fieldConfig": {
            "defaults": {
              "custom": {}
            },
            "overrides": []
          },
          "fill": 1,
          "fillGradient": 0,
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 36
          },
          "hiddenSeries": false,
          "id": 65,
          "legend": {
            "alignAsTable": true,
            "avg": false,
            "current": false,
            "hideEmpty": true,
            "max": true,
            "min": false,
            "rightSide": true,
            "show": true,
            "total": false,
            "values": true
          },
          "lines": true,
          "linewidth": 1,
          "nullPointMode": "null",
          "options": {
            "alertThreshold": true
          },
          "percentage": false,
          "pluginVersion": "7.3.0",
          "pointradius": 2,
          "points": false,
          "renderer": "flot",
          "scopedVars": {
            "infra": {
              "selected": true,
              "text": "ip-10-0-152-106.us-west-2.compute.internal",
              "value": "ip-10-0-152-106.us-west-2.compute.internal"
            }
          },
          "seriesOverrides": [
            {
              "$$hashKey": "object:1826",
              "alias": "/rxDroppedPackets.*/",
              "transform": "negative-Y"
            }
          ],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [
            {
              "alias": "txDroppedPackets - {{labels.device.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:1751",
                  "fake": true,
                  "field": "labels.device.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": 0,
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:1370",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": 0,
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "hide": false,
              "metrics": [
                {
                  "$$hashKey": "object:1368",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName: \"txDroppedPackets\" AND labels.instance.keyword: $infra",
              "refId": "A",
              "timeField": "timestamp"
            },
            {
              "alias": "rxDroppedPackets - {{labels.device.keyword}}",
              "bucketAggs": [
                {
                  "$$hashKey": "object:1768",
                  "fake": true,
                  "field": "labels.device.keyword",
                  "id": "3",
                  "settings": {
                    "min_doc_count": 0,
                    "order": "desc",
                    "orderBy": "_term",
                    "size": "10"
                  },
                  "type": "terms"
                },
                {
                  "$$hashKey": "object:1594",
                  "field": "timestamp",
                  "id": "2",
                  "settings": {
                    "interval": "30s",
                    "min_doc_count": 0,
                    "trimEdges": 0
                  },
                  "type": "date_histogram"
                }
              ],
              "hide": false,
              "metrics": [
                {
                  "$$hashKey": "object:1592",
                  "field": "value",
                  "id": "1",
                  "meta": {},
                  "settings": {},
                  "type": "avg"
                }
              ],
              "query": "uuid: $uuid AND metricName: \"rxDroppedPackets\" AND labels.instance.keyword: $infra",
              "refId": "B",
              "timeField": "timestamp"
            }
          ],
          "thresholds": [],
          "timeFrom": null,
          "timeRegions": [],
          "timeShift": null,
          "title": "Dropped packets $infra",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "$$hashKey": "object:1701",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "$$hashKey": "object:1702",
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ],
          "yaxis": {
            "align": false,
            "alignLevel": null
          }
        }
      ],
      "repeat": "infra",
      "title": "Infra: $infra",
      "type": "row"
    }
  ],
  "refresh": false,
  "schemaVersion": 26,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": [
      {
        "current": {
          "selected": false,
          "text": "ripsaw-kube-burner-internal-production",
          "value": "ripsaw-kube-burner-internal-production"
        },
        "error": null,
        "hide": 0,
        "includeAll": false,
        "label": "Datasource",
        "multi": false,
        "name": "Datasource",
        "options": [],
        "query": "elasticsearch",
        "queryValue": "",
        "refresh": 1,
        "regex": "/.*kube-burner.*/",
        "skipUrlSync": false,
        "type": "datasource"
      },
      {
        "allValue": null,
        "current": {},
        "datasource": "$Datasource",
        "definition": "{\"find\": \"terms\", \"field\": \"uuid.keyword\"}",
        "error": null,
        "hide": 0,
        "includeAll": false,
        "label": "UUID",
        "multi": false,
        "name": "uuid",
        "options": [],
        "query": "{\"find\": \"terms\", \"field\": \"uuid.keyword\"}",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {},
        "datasource": "$Datasource",
        "definition": "{ \"find\" : \"terms\", \"field\": \"labels.node.keyword\", \"query\": \"metricName.keyword: nodeRoles AND labels.role.keyword: master AND uuid.keyword: $uuid\"}",
        "error": null,
        "hide": 0,
        "includeAll": true,
        "label": "Master nodes",
        "multi": true,
        "name": "master",
        "options": [],
        "query": "{ \"find\" : \"terms\", \"field\": \"labels.node.keyword\", \"query\": \"metricName.keyword: nodeRoles AND labels.role.keyword: master AND uuid.keyword: $uuid\"}",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {},
        "datasource": "$Datasource",
        "definition": "{ \"find\" : \"terms\", \"field\": \"labels.node.keyword\", \"query\": \"metricName.keyword: nodeRoles AND labels.role.keyword: worker AND uuid.keyword: $uuid\"}",
        "error": null,
        "hide": 0,
        "includeAll": false,
        "label": "Worker nodes",
        "multi": true,
        "name": "worker",
        "options": [],
        "query": "{ \"find\" : \"terms\", \"field\": \"labels.node.keyword\", \"query\": \"metricName.keyword: nodeRoles AND labels.role.keyword: worker AND uuid.keyword: $uuid\"}",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {},
        "datasource": "$Datasource",
        "definition": "{ \"find\" : \"terms\", \"field\": \"labels.node.keyword\",  \"query\": \"metricName.keyword: nodeRoles AND labels.role.keyword: infra AND uuid.keyword: $uuid\"}",
        "error": null,
        "hide": 0,
        "includeAll": false,
        "label": "Infra nodes",
        "multi": true,
        "name": "infra",
        "options": [],
        "query": "{ \"find\" : \"terms\", \"field\": \"labels.node.keyword\",  \"query\": \"metricName.keyword: nodeRoles AND labels.role.keyword: infra AND uuid.keyword: $uuid\"}",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      },
      {
        "allValue": null,
        "current": {},
        "datasource": "$Datasource",
        "definition": "{ \"find\" : \"terms\", \"field\": \"labels.namespace.keyword\", \"query\": \"labels.namespace.keyword: /openshift-.*/ AND uuid.keyword: $uuid\"}",
        "error": null,
        "hide": 0,
        "includeAll": true,
        "label": "Namespace",
        "multi": true,
        "name": "namespace",
        "options": [],
        "query": "{ \"find\" : \"terms\", \"field\": \"labels.namespace.keyword\", \"query\": \"labels.namespace.keyword: /openshift-.*/ AND uuid.keyword: $uuid\"}",
        "refresh": 2,
        "regex": "",
        "skipUrlSync": false,
        "sort": 0,
        "tagValuesQuery": "",
        "tags": [],
        "tagsQuery": "",
        "type": "query",
        "useTags": false
      }
    ]
  },
  "time": {
    "from": "2020-11-23T09:37:39.828Z",
    "to": "2021-07-21T09:37:39.836Z"
  },
  "timepicker": {
    "refresh_intervals": [
      "5s",
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ]
  },
  "timezone": "",
  "title": "Kube-burner report",
  "uid": "hIBqKNvMz",
  "version": 70
}
```
