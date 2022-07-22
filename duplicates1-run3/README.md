# Testing for duplicates

Logs corresponding to test run
https://github.com/knative-sandbox/eventing-kafka-broker/issues/2240#issuecomment-1191831657

## Deploy Kafka Source

To reproduce setup everything needed is to deploy Knative Kafka Source 1.6:

```
k apply -f https://github.com/knative-sandbox/eventing-kafka-broker/releases/download/knative-v1.6.0/eventing-kafka-source-bundle.yaml
```

## Configure Kafka Source

### Edit config to enable rate-limiting and executor queue metrics

Defined in

control-plane/config/eventing-kafka-source/200-controller/100-config-kafka-features.yaml


Change:

```
  dispatcher.rate-limiter: "enabled"
  dispatcher.ordered-executor-metrics: "enabled"
```

```
k apply -f control-plane/config/eventing-kafka-source/200-controller/100-config-kafka-features.yaml
```

Verify

```
k -n knative-eventing get cm config-kafka-features -oyaml | grep enabled
```


### Edit config to use CooperativestickyAssignor and change max.partition.fetch.bytes and auto.commit


Defined in

data-plane/config/source/100-config-kafka-source-data-plane.yaml

Change:

```
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

```
#max.partition.fetch.bytes=1048576
max.partition.fetch.bytes=128
```

```
#auto.commit.interval.ms=5000
auto.commit.interval.ms=500
```


```
k apply -f data-plane/config/source/100-config-kafka-source-data-plane.yaml
```

Verify

```
k -n knative-eventing get cm config-kafka-source-data-plane -oyaml |grep partition.assignment.strategy
```

```
k -n knative-eventing get cm config-kafka-source-data-plane -oyaml |grep max.partition.fetch.bytes
```

```
k -n knative-eventing get cm config-kafka-source-data-plane -oyaml |grep auto.commit.interval.ms
```

## Test Kafka Source

To gather logs about Knative Kafka Source in separate windows run

Knative eventing data plane

```
k -n knative-eventing logs -l=app=kafka-source-dispatcher --max-log-requests 10 -f | tee logs-kafka-source-data-plane4.txt
```

Knative eventing controller


```
k -n knative-eventing logs -l=app=kafka-source-controller -f  | tee logs-kafka-source-controller4.txt
```


To run tests

```
git clone https://github.com/aslom/reproduce-kafka-source.git
cd reproduce-kafka-source
export KO_DOCKER_REPO=...
ko apply -f duplicates1
```


Logs for test side:

```
k logs -l=app=eventdisplay -f | tee logs-event-display4.txt
```


```
k logs -l=app=kafkascraper -f | tee logs-kafkaproducer4.txt
```


After one minute running the test scale Kafka source

```
kubectl scale --replicas=50 kafkasources/kafka-src50
```

To monitor test exeuciton with stats:

```
kubectl run curl --image=curlimages/curl --rm=true --restart=Never -ti -- http://kafkascraper/stats
```

Expected output should look like

```
source kafka-src50 received: 8055 messages, 0 are dups and 0 non 200s.
```


And to check how many messages are consumed form test topic paritiions (look for LAG):

```
kubectl -n kafka run kafka-consumer-groups -ti --image=quay.io/strimzi/kafka:0.28.0-kafka-3.1.0 --rm=true --restart=Never -- bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe  --group kafka-src50c
```

