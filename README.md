# Intro - Guide HowTo Monitoring and Logging OCP4 to ElasticSearch
In Openshift environments, as part of the overall Observability field, it is crucial to implement solutions in two areas: Monitoring and Logging.


## Monitoring vs. Logging

The purpose of _Logging_ is to create an ongoing record of application events.

In case of Openshift - logging are relevant both for logs from the Openshift Platform itself (i.e. Infrastructure & Adit logs) and logs from the apps running on top of openshift (Application Logs).

_Monitoring_ is an umbrella term related to gathering information (Metrics) about system performance and applications' resource usage accross time.

In case of Openshift, yet again, monitoring are relevant for both Platform (Openshift) metrics and Application metrics for the applications that run on top of Openshift.


## Do we really need both?
**Yes!**

Logs and Metrics are both used to analyze how our system works - in terms of applicaion logs that allow us to know what process is currently running and what the next phase is going to be as part of the application flow, and in term of metrics we can understand our application resources consumption at any given moment. 

By using both of these Obesrvability methodologies, we are covering all aspects of the applicationthese Obesrvability methodologies, we are covering almost every aspect of the application - for troubleshooting purposes and adjusting the application to perform better in the future.


# Technologies we are going to use
## Logs & Metrics Collector - ElasticSearch
To be the most efficiat we want to collect all the metrics and logs to one place so we could access them from one product.

ElasticSearch is one of the leading tools to collect big amounts of data from verious sources and it allows you to `index` the data in such manner that you'll be able to filter it later on when you will need it.

We will see in this guide how to configure the different indexes for the different sources of logs and metrics that we will create.

## Gathering Logs from Openshift - Openshift Logging Operator (Fluentd)
The Openshift Logging Operator is an extention of Openshift (proivisioned & supported by Red Hat) that allow us to deploy privileged, protected, `Fluentd` pods on each and every node in our cluster, and send it later to Elastic.

The choise of using `Fluentd` is not random at all. Fluentd is an extreamly flexiable solution to gather logs from multiple input sources, and send the logs with different tags to various output locations. In such a way we can send logs tagged as "Infra", "Audit" and "App" from the same Openshift cluster, to the same Elastic, and use these logs in an easier manner later on for troubleshooting or whatever we want. 

We will see later in the guide how to configure Fluentd to tag the logs differently.

## Gathering Metrics from Openshift - Openshift Monitoring (Prometheus)
The Openshift Monitoring Operator is the main source of truth for all collected metrics in an Openshift cluster. 

This operator comes built-in with Openshift 4 upon installation, and by default it collects infrastructure metrics from Openshift. This Red Hat Operator allow us to extand its capabilities - we can configure it to also collect user applications metrics by deploying another Prometheus instance dedicated for that purpus.

We will see later on how we are extending the built-in operator and sending all the metrics from both Prometheuses to ElasticSearch

## Last but not least - Metricbeat
Metricbeat is a utility tool created by the same company of ElasticSearch that make it easier to send metrics from metric-stoiring systems such as Prometheus to Elasticsearch and we will use it as well.

---

# Let's get technical

## Starting with ElasticSearch
I must admit that ElasticSearch is not my specialty and that in most organizations ElasticSearch is a tool managed by a dedicated team because of its complexity. 

In this guide I'm taking into consideration that there is an ElasticSearch instance already installed.

### Configuring Retention/Rollouver Policy, Template, and Index
In ElasticSearch we are going to create for each source of logs/metrics three objects:
1. Retention/Rollover Policy - allow us to determin for how long Elastic should store the data before it starts to delete the old data gathered.
2. Template & Index - the actuall index the data is going to be written to. Template is the general object that we will configure the gathering tool (Prometheus/Fluentd) to send the data to, and index is the first "data-block" that the data is going to be written to. Once the Index "data-block" is filled to the maximum, Elasticsearch will generate the next one automaticly.


#### 1 Login to Elasticsearch
Login to the Elasticsearch URL with the **port 5601** and fill the username + password.

Next, navigate to the `Dev Tools` page.

#### 2 Generate Retention Policy
In our case we are going to use the same retention/rollover policy for all of our indexes.

In the `Dev Tools` page, enter the following GET request to see all the available Policies:
```bash
GET _ilm/policy
```

Generate the policy in case it is missing by pasting the following lines in the same `Dev Tools` page:
> Change the CHANGEME to the amount of days you want. Example: For 3 days, change it to ```3d```
> Notice that I've mentioned `pocp` in the name of the policy. I did it because Elasticsearch can gather data from multiple clusters and the rollover policies might be different for prod-ocp (pocp) and dev-ocp (docp)
```bash
PUT _ilm/policy/ocp_pocp_rollover_policy_CHANGEME
{
  "policy" : {
    "phases" : {
      "hot" : {
        "min_age" : "0ms",
        "actions" : {
          "rollover" : {
             "max_size" : "20gb",
             "max_age": "12h"
          },
          "set_priority" : {
            "priority" : 100
          }
        }
      },
      "delete" : {
        "min_age" : "CHANGEME",
        "actions" : {
          "delete" : {}
        }
      }
    }
  }
}
```

Run the following again and see that the policy exists:
```bash
GET _ilm/policy
```

#### 3 Generate Template
For each index required, generate a template. In our case we are about to generate 5:
1. Infrastructure Metrics (Prometheus + Metricbeat)
2. Application Metrics (Promtheus + Metricbeat)
3. Infrastructure Logs (Fluentd)
4. Audit Logs (Fluentd)
5. Application Logs (Fluentd)

To see all available templates, run the following in the `Dev Tools` Page:
> Notice that I'm mentioning `pocp` in the names of the templates because the same Elastic can gather data from multiple clusters
```bash
GET _template/ocp-*-write-template
```
To generate the 5 templates, run the following:
1. Infrastructure Metrics
```bash
PUT _template/ocp-pocp-metrics-infra-write-template
{
  "order" : 0,
  "index_patterns" : [
    "ocp-pocp-metrics-infra-write*"
  ],
  "settings" : {
    "index" : {
      "lifecycle" : {
        "name" : "ocp_pocp_rollover_policy_CHANGEME",
        "rollover_alias" : "ocp-pocp-metrics-infra-write"
      },
      "mapping" : {
        "total_fields" : {
          "limit": "5000"
        }
      },
      "number_of_shards" : "1",
      "number_of_replicas" : "1"
    }
  },
  "mappings" : { },
  "aliases" : { }
}
```
2. Application Metrics
```bash
PUT _template/ocp-pocp-metrics-app-write-template
{
  "order" : 0,
  "index_patterns" : [
    "ocp-pocp-metrics-app-write*"
  ],
  "settings" : {
    "index" : {
      "lifecycle" : {
        "name" : "ocp_pocp_rollover_policy_CHANGEME",
        "rollover_alias" : "ocp-pocp-metrics-app-write"
      },
      "mapping" : {
        "total_fields" : {
          "limit": "5000"
        }
      },
      "number_of_shards" : "1",
      "number_of_replicas" : "1"
    }
  },
  "mappings" : { },
  "aliases" : { }
}
```
3. Infrastructure Logs 
```bash
PUT _template/ocp-pocp-logs-infra-write-template
{
  "order" : 0,
  "index_patterns" : [
    "ocp-pocp-logs-infra-write*"
  ],
  "settings" : {
    "index" : {
      "lifecycle" : {
        "name" : "ocp_pocp_rollover_policy_CHANGEME",
        "rollover_alias" : "ocp-pocp-logs-infra-write"
      },
      "mapping" : {
        "total_fields" : {
          "limit": "5000"
        }
      },
      "number_of_shards" : "1",
      "number_of_replicas" : "1"
    }
  },
  "mappings" : { },
  "aliases" : { }
}
```
4. Audit Logs 
```bash
PUT _template/ocp-pocp-logs-audit-write-template
{
  "order" : 0,
  "index_patterns" : [
    "ocp-pocp-logs-audit-write*"
  ],
  "settings" : {
    "index" : {
      "lifecycle" : {
        "name" : "ocp_pocp_rollover_policy_CHANGEME",
        "rollover_alias" : "ocp-pocp-logs-audit-write"
      },
      "mapping" : {
        "total_fields" : {
          "limit": "5000"
        }
      },
      "number_of_shards" : "1",
      "number_of_replicas" : "1"
    }
  },
  "mappings" : { },
  "aliases" : { }
}
```
5. Application Logs 
```bash
PUT _template/ocp-pocp-logs-app-write-template
{
  "order" : 0,
  "index_patterns" : [
    "ocp-pocp-logs-app-write*"
  ],
  "settings" : {
    "index" : {
      "lifecycle" : {
        "name" : "ocp_pocp_rollover_policy_CHANGEME",
        "rollover_alias" : "ocp-pocp-logs-app-write"
      },
      "mapping" : {
        "total_fields" : {
          "limit": "5000"
        }
      },
      "number_of_shards" : "1",
      "number_of_replicas" : "1"
    }
  },
  "mappings" : { },
  "aliases" : { }
}
```

Run the same validation command to see all templates you just created:
```bash
GET _template/ocp-*-write-template
```


#### 4 Generate the first Index for each template
Run the following commands in the `Dev Tools` to generate the first index the the 5 templates we generated above:
> Again, there might be a few Openshift clusters sending data to this Elasticsearch, make sure you are generating the indexes & templates with the reference to the right cluster (pocp == prod ocp, dev-ocp, etc.)
1. Infrastructure Metrics
```bash
PUT ocp-pocp-metrics-infra-write-000001
{
  "aliases" :
  {
    "ocp-pocp-metrics-infra-write" : {}
  }
}
```
2. Application Metrics
```bash
PUT ocp-pocp-metrics-app-write-000001
{
  "aliases" :
  {
    "ocp-pocp-metrics-app-write" : {}
  }
}
```
3. Infrastructure Logs
```bash
PUT ocp-pocp-logs-infra-write-000001
{
  "aliases" :
  {
    "ocp-pocp-logs-infra-write" : {}
  }
}
```
4. Audit Logs
```bash
PUT ocp-pocp-logs-audit-write-000001
{
  "aliases" :
  {
    "ocp-pocp-logs-audit-write" : {}
  }
}
```
5. Application Logs
```bash
PUT ocp-pocp-logs-app-write-000001
{
  "aliases" :
  {
    "ocp-pocp-logs-app-write" : {}
  }
}
```

Run the following command to see all the indexes you just created in Elasticsearch:
```bash
GET _cat/indices/ocp-*-write*?v&s=i
```

And finally, test the rollout:
```bash
POST ocp-pocp-logs-app-write\_rollover
POST ocp-pocp-logs-audit-write\_rollover
POST ocp-pocp-logs-infra-write\_rollover
POST ocp-pocp-metrics-infra-write\_rollover
POST ocp-pocp-metrics-app-write\_rollover
```
---

## Sending Metrics - Openshift, Prometheus, Metricbeat
As mentioned in the intro, there are two types of metrics - infra metrics and application metrics, each is collected / scraped by a different Prometheus in Openshift.

### Infrastructure Metrics
We are not going to use prometheus to send these metrics to Elasticsearch, but rather **Metricbeat**.

We are going to install this Metricbeat as DaemonSet on all of our nodes with privileged permissions so it will be able to collect the metrics from each node directly.

I took the original Metricbeat provided by Elasticsearch and rewrite it as `helm chart` for easier installation.

We are also using two special modules called `Kubernetes` and `System` so metricbeat will know what it should gather from the system.

To install my helm chart you'll need to clone this repo and run the following:
> Edit the relevant variables in the [values.yaml](./metricbeat-infra/values.yaml) file in the chart so it'll match your environment.
```bash
oc create ns openshift-metricbeat-infra-daemonset
oc project openshift-metricbeat-infra-daemonset
oc adm policy add-scc-to-user privileged -z metricbeat -n openshift-metricbeat-infra-daemonset
cd metricbeat-infra
helm install metricbeat-infra . -f values.yaml -n openshift-metricbeat-infra-daemonset
```

Verification:
```bash
helm list
oc get pods -n openshift-metricbeat-infra-daemonset
```

Next, go to elasticsearch and run the following in the `Dev Tools` Page; The counter should increas as the metrics are written.
```bash
GET _cat/indices/ocp-*-metrics-infra-write*?v&s=i
```

### User-Workloads (App) Metrics
First, we need to extend the built-in Openshift-Monitoring operator to provide Prometheus for scraping metrics from user applications;

Run the following in Openshift as logged-in `cluster-admin` user:
```bash
oc -n openshift-monitoring edit configmap cluster-monitoring-config
```
It should contain the following lines:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true 
```
You should be able to see that Prometheus user-workload has been deployed:
```bash
oc get pods -n openshift-user-workload-monitoring
```

Now, we will deploy a standard non-privileged Metricbeat instance, and configure Prometheus to send metrics to this Metricbeat with the `Remote_write` feature;

#### Deploy metricbeat
First, build the Dockerfile in the directory [metricbeat-deployment-image](./metricbeat-deployment-image) with instructions inside

Then, go to the [metricbeat-user-workloads](./metricbeat-user-workloads/) directory and run the following commands:
> NOTE! Edit the [values.yaml](./metricbeat-user-workloads/values.yaml) file before you install the helm chart to correspond with your values
```bash
helm install metricbeat-user-workload . -f values.yaml -n openshift-user-workload-monitoring
```

Verify
```bash
helm list
oc get pods -n openshift-user-workload-monitoring
```

This is not enough, we have one more step - configure Prometheus (User-workloads) to send (`remote_write`) the metrics to the metricbeat deployment we've just installed:

```bash
oc edit cm user-workload-monitoring-config -n openshift-user-workload-monitoring
```
Make sure it has the following lines:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-workload-monitoring-config
  namespace: openshift-user-workload-monitoring
data:
  config.yaml: |
    prometheus:
      remoteWrite:
        - url: "http://metricbeat.openshift-user-workload-monitoring.svc.cluster.local:9201/write"
          tlsConfig:
            insecure_skip_verify: true
```
Now you need to scale the existing prometheus pods down to 0, don't worry due, the Openshift-Monitoring operator will generate two new instances for you automaticly, and they will have the new configuration we just created.
```bash
oc scale --replicas=0 statefulset prometheus-user-workload -n openshift-user-workload-monitoring
# Optional if you don't want to wait
oc scale --replicas=2 statefulset prometheus-user-workload -n openshift-user-workload-monitoring
```

Finaly, go to Elasticsearch -> `Dev Tools` page and test the index:
```bash
GET _cat/indices/ocp-*-metrics-app-write*?v&s=i
```

## Sending Logs - Fluentd - Infra, Audit & App Logs
We need to do several things to make it work:
0. Create Namespace `openshift-logging` and label it
1. Install Openshift Logging Operator + Elasticsearch operator via OperatorHub
2. Generate Clusterlogging instance
3. Change the Clusterlogging instance as `Unmanaged` 
> We are going to disable Kibana and Elasticsearch that the operator usually install for us because we assumed that we already have central elasticsearch for this organization
4. Delete the elasticsearch and kibana that the operator generates for you
5. Configure ephemeral storage quota to prevent over-consumption of ephmeral storage (emptyDir) on the nodes
6. Edit the Fluentd configuration file to tag the data with the relevant tag and send it to the right index in our extranal Elasticsearch

### 0 Create Namespace + Label it
```bash
oc create namespace openshift-logging
# Red Hat Solution #6165332
oc label namespace openshift-logging openshift.io/cluster-monitoring="true"
```
### 1 Install Openshift Logging Operator
Via Openshift OperatorHub. 
> In case of disconnected network, you'll need to import it to Openshift. 
### 2 Generate Clusterlogging instance
Via Openshift Dashboard => Installed Operators => Openshift Logging => ClusterLogging => Create New Instance (leave all the default values)
### 3 Change the operator to Unmanaged state
```bash
cat > clusterloggingUnmanaged.yaml << EOF
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  collection:
    logs:
      fluentd: {}
      type: fluentd
  curation:
    curator:
      schedule: 30 3 * * *
    type: curator
  logStore:
    elasticsearch:
      nodeCount: 0
      redundancyPolicy: ZeroRedundancy
      storage: {}
    retentionPolicy: {}
    type: elasticsearch
  managementState: Unmanaged
  visualization:
    kibana:
      replicas: 0
    type: kibana
EOF

oc apply -f clusterloggingUnmanaged.yaml
```
### 4 Delete the elasticsearch and kibana that the operator generates for you
```bash
oc delete elasticsearch --all -n openshift-logging
oc delete kibana --all -n openshift-logging
```

### 5 Configure ephemeral storage quota
```bash
cat > ephemeral-storage-quota.yaml << EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: openshift-logging-ephemeral-storage-quota
  namespace: openshift-logging
spec:
  hard:
    limits.ephemeral-storage: 4Gi
EOF

oc apply -f ephemeral-storage-quota.yaml
```

### 6 Apply Fluentd Configuration File + kill existing pods
The fluentd configuration file is really tedious.

I generated an Helm chart that will help you easly adjust the configuration file to your needs, without being familiar with the entire file. 

To adjust the Helm chart to match your needs, edit the [values.yaml](./fluentd/values.yaml) file in the helm chart
---
There are several key elements useful to get familiar with when deploying this fluentd configuration file, you can see them all in [THIS FILE](./fluentd/templates/fluentd-cm.yaml);

### i. Link the different log sources to Elasticsearch dedicated index
> Infra Logs
```xml
<elasticsearch_index_name>
  enabled 'true'
  tag "journal.system** system.var.log** **_default_** **_kube-*_** **_openshift-*_** **_openshift_**"
  name_type static
  static_index_name {{ .Values.infra_logs_index }}
</elasticsearch_index_name>
```
> Audit Logs
```xml
<elasticsearch_index_name>
  enabled 'true'
  tag "linux-audit.log** k8s-audit.log** openshift-audit.log** ovn-audit.log**"
  name_type static
  static_index_name {{ .Values.audit_logs_index }}
</elasticsearch_index_name>
```
> Application Logs
```xml
<elasticsearch_index_name>
  enabled 'true'
  tag "kubernetes.**"
  name_type static
  static_index_name {{ .Values.app_logs_index }}
</elasticsearch_index_name>
```

### ii. Relabel the different log sources
```xml
  # Include Infrastructure logs
  <match **_default_** **_kube-*_** **_openshift-*_** **_openshift_** journal.** system.var.log**>
    @type relabel
    @label @_INFRASTRUCTURE
  </match>
  
  # Include Application logs
  <match kubernetes.**>
    @type relabel
    @label @_APP
  </match>
  
  # Include Audit logs
  <match linux-audit.log** k8s-audit.log** openshift-audit.log** ovn-audit.log**>
    @type relabel
    @label @_AUDIT
  </match>
```

### iii. Relabel different types to be forwarded to Elasticsearch
```xml
# Sending audit source type to pipeline
<label @_AUDIT>
  <filter **>
    @type record_modifier
    <record>
      log_type audit
    </record>
  </filter>
  
  <match **>
    @type relabel
    @label @FORWARD_TO_REMOTE
  </match>
</label>

# Sending app source type to pipeline
<label @_APP>
  <filter **>
    @type record_modifier
    <record>
      log_type app
    </record>
  </filter>
  
  <match **>
    @type relabel
    @label @FORWARD_TO_REMOTE
  </match>
</label>

# Copying pipeline forward-to-remote to outputs
<label @FORWARD_TO_REMOTE>
  <match **>
    @type relabel
    @label @REMOTE_ELASTICSEARCH
  </match>
</label>
```

### iv. Configure the Elasticsearch login details to be used by fluentd upon forwarding
```xml
  <match retry_remote_elasticsearch>
    @type elasticsearch
    @id retry_remote_elasticsearch
    host {{ .Values.host }}
    port {{ .Values.port }}
    user {{ .Values.user }}
    password {{ .Values.password }}
    ...
    </buffer>
  </match>
  
  <match **>
    @type elasticsearch
    @id remote_elasticsearch
    host {{ .Values.host }}
    port {{ .Values.port }}
    user {{ .Values.user }}
    password {{ .Values.password }}
    ...
    </buffer>
  </match>
```
---
To apply this configuration, run the following command when inside the [fluentd directory](./fluentd/)
```bash
helm install fluentd-configuration . -f values.yaml
```

And finally, delete the existing fluentd pods. The operator will automaticly generate new pods that will have the new configuration

```bash
oc delete pods -l=component=collector -n openshift-logging
```


