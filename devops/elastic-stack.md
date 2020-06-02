---
description: 'Logging Architecture - Formerly ELK - Elasticsearch, Logstash, Kibana'
---

# Elastic stack

The Elastic stack \(formerly ELK stack\) is a collection of three popular open-source projects that are now widely used collectively for logging: Elasticsearch, Logstash, and Kibana.

_Logstash_ is used for collecting and filtering the input data. The filtered data will then be stored in Elasticsearch. _Elasticsearch then_ indexes the stored data for quick searches. Finally, _Kibana_ lets us collect, visualize, and analyze our Elasticsearch data. Kibana's histograms, line graphs, pie charts, sunbursts leverage the full aggregation capabilities of Elasticsearch.

### Getting Started

The Elastic stack is most commonly installed on Linux or other Unix-based systems, but this guide is about adding logging capabilities to your service. Containerized applications are a bit different, so as a pre-requisite, it assumes that you have followed the instructions on setting up your localdev and you're familiar with adding a new service to your cluster.

The end result will be three containers running in parallel - one for each, with port forwarding setup for Kibana.

### Setting up Elasticsearch

_**Elasticsearch**_ is the component responsible for storing and indexing data collected from the application, hence, it's the easiest to set up. 

Just follow the step for adding a service in localdev and make sure its identifiers are named _'elasticsearch',_ and that it uses **port** **9200** \(the default port for Elasticsearch\),  in deployment and service yaml templates.

After that, the only things necessary to have are the container image URL and the _imagePullSecrets_, so your entire Elasticsearch '_values.yml_ 'should just look like:

```text
container:
  image: open.docker.ing.net/dbnl/demo/elasticsearch:6.4.2

imagePullSecrets:
- regsecret
```

### Setting up Logstash

_**Logstash**_ collects, filters and _stashes_ app data into Elasticsearch. In order to set it up, we will have to do a bit more work to the generic localdev chart template. But first, follow the steps in adding a service just like with elasticsearch - but this time, use __name _'logstash'_ with **port 12200** in the ****corresponding deployment and service templates**.** Similarly, the _values.yml_ should reflect the container image location of logstash as such:

```text
container:
  image: open.docker.ing.net/dbnl/demo/logstash:6.4.2

imagePullSecrets:
- regsecret
```

Now remember that logstash collects data and send them to the Elasticsearch 'database', thus we also have to set up where it's going to get application log data from, and where it should send them to. We do this in the _configmap.yml_ file inside the templates folder of your helm chart.

We defined elasticsearch service to use port 9200, thus in c_onfigmap.yml,_ the output variable should reflect this. As such:

```text
apiVersion: v1
kind: ConfigMap
metadata:
  name: logstash-cm
data:
  logstash-kafka.conf: |-
    input {
        kafka {
                bootstrap_servers => "localdev-kafka:9092"
                topics => ["log_globtpa_tech_topic"]
        }
    }

    output {
       elasticsearch {
          hosts => ["http://elasticsearch:9200"]
          index => "log_globtpa_tech_topic"
          workers => 1
        }
    }

```

If you've noticed, the input variable uses Kafka. It is the most common buffer solution deployed together the **ELK** Stack and it is closely tied with the application you're trying to get the logs from.

### Setting up Kafka

For brevity, it is assumed that the application is developed using spring boot / ING merak.

In order to use Kafka, it must be downloaded as a dependency defined in the pom.xml of your spring application. ING provides its own Kafka artifact that you should use.

```text
 <dependency>
    <groupId>com.ing.log</groupId>
    <artifactId>logback-kafka-appender</artifactId>
 </dependency>
```

And in order to enable the logging from your application, this flag must be set to true in your application.properties file:

```text
merak.tracing.reporting.kafka.enabled=true
```

The above steps ensure that Kafka will be able to receive the logs from your application, however, your application must still have a way of logging like `insurance-be-product`'s LogWatcher class because Kafka will only handle the buffering of log stream to Logstash.

### Setting up Kibana

For Kibana, we repeat the copy generic helm chart procedure from adding a service. However, the names shoud be _kibana_ and using **port 5601.** Likewise, the values.yml should be \(but with an added variable source of data it will use for visualization\):

```text
env:
  elasticsearch.url: http://elasticsearch:9200
  xpack.security.enabled: "false"

container:
  image: open.docker.ing.net/dbnl/demo/kibana:6.4.2

imagePullSecrets:
- regsecret
```

### Accessing the Kibana dashboard

Make sure your application and the three ELK services are up and running \(maybe screenshot of oc get po? Zookeper?\). 

Then go to http://kibana:5601
