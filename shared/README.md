# Shared
## Overview

In this example I configure a Local Confluent Control Center (C3) to monitor a Confluent Cloud (basic)
Cluster. I then configure a `console` based producer and a `console` based consumer using Confluent 
Monitoring Interceptors that report to the same Confluent Cloud Basic Cluster

## Prereqs
- running cluster in the cloud
- payload data and metrics data hosted in the same cluster
(-I'm using v 7.1.1 of the Confluent Platform)


## Steps

* configure and start a local C3, using Metrics data cluster
* configure and run a producer, with monitoring interceptors
* configure and run a consumer, with monitoring interceptors 


## configure and start a local C3
I'm using `bootstrap server` and a _Cluster_ API key/Secret pair for my *_metrics cluster_*
I'm adjusting an existing basic control-center-properties file and set the minimum target and authentication
properties as per step 4 in https://docs.confluent.io/cloud/current/cp-component/c3-cloud-config.html

```
bootstrap.servers=[BOOTSTRAP_SERVER URL HERE]
confluent.controlcenter.streams.security.protocol=SASL_SSL
confluent.controlcenter.streams.sasl.mechanism=PLAIN
confluent.controlcenter.streams.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="[CLUSTER API KEY]" \
  password="[CLUSTER API SECRET]";
confluent.metrics.topic.max.message.bytes=8388608
confluent.controlcenter.streams.ssl.endpoint.identification.algorithm=https
```

use your confluent platform install `bin` directory to start controlcenter locally with the controlcenter properties

```
/path/to/confluent-7.1.1/bin/control-center-start control-center.properties
```

Verify success by logging onto `http://localhost:9021`, create a topic "cc-metrics-demo" (with defaults) and observe that the 
topic "cc-metrics-demo" appears in the confluent cloud console of your metrics cluster


## produce data to payload cluster, report metrics to metrics cluster

Produce settings in producer.properties
```
bootstrap.servers=[BOOTSTRAP_SERVER URL HERE]
security.protocol=SASL_SSL
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule   required \
  username="[CLUSTER API KEY]" \
  password="[CLUSTER API SECRET]";
sasl.mechanism=PLAIN

client.id=cc-metrics-demo-producer

linger.ms=1000

```


Interceptor settings in producer.properties
```
interceptor.classes=io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor
confluent.monitoring.interceptor.security.protocol=SASL_SSL
confluent.monitoring.interceptor.sasl.mechanism=PLAIN
confluent.monitoring.interceptor.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="[CLUSTER API KEY]" \
  password="[CLUSTER API SECRET]";
confluent.monitoring.interceptor.ssl.endpoint.identification.algorithm=https
```


example console producer
```
export CLASSPATH=/path/to/confluent-7.1.1/share/java/monitoring-interceptors/monitoring-interceptors-7.1.1.jar
for x in {1..5000}; \
do echo $x; \
sleep 0.02; \
done | kafka-console-producer \
--bootstrap-server [BOOTSTRAP_SERVER URL HERE]  \
--topic cc-metrics-demo \
--producer.config producer.properties
```

## Consumer side

```
bootstrap.servers=[BOOTSTRAP_SERVER URL HERE]
security.protocol=SASL_SSL
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule   required \
  username="[CLUSTER API KEY]" \
  password="[CLUSTER API SECRET]";
sasl.mechanism=PLAIN

client.id=cc-metrics-demo-consumer
group.id=cc-metrics-demo-consumer-group
```

```
interceptor.classes=io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor
confluent.monitoring.interceptor.bootstrap.servers=[BOOTSTRAP_SERVER URL HERE]
confluent.monitoring.interceptor.security.protocol=SASL_SSL
confluent.monitoring.interceptor.sasl.mechanism=PLAIN
confluent.monitoring.interceptor.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="[CLUSTER API KEY]" \
  password="[CLUSTER API SECRET]";
confluent.monitoring.interceptor.ssl.endpoint.identification.algorithm=https
```


```
export CLASSPATH=/path/to/confluent-7.1.1/share/java/monitoring-interceptors/monitoring-interceptors-7.1.1.jar
kafka-console-consumer \
--bootstrap-server [BOOTSTRAP_SERVER URL HERE]  \
--topic cc-metrics-demo \
--consumer.config consumer.properties
```

