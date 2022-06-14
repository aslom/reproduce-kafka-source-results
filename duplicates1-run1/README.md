Logs corresponding to test run
https://github.com/knative-sandbox/eventing-kafka-broker/issues/2240#issuecomment-1151455968

To reproduce setup everything needed for Knative and then:

```
https://github.com/knative-sandbox/eventing-kafka-broker.git
cd eventing-kafka-broker
git checkout 3287ed6dcf949032bb805652e504b9c03ae472ab
./hack/run.sh deploy-source
```

To run

```
git clone https://github.com/aslom/reproduce-kafka-source.git
cd reproduce-kafka-source
# optional - if you want to run exactlu the smae test code:
#git checkout 1abba8978e8694e16ebc187c6a0211f89c03ea2b
ko apply -f duplicates1
``

To scale Kafka source

```
kubectl scale --replicas=50 kafkasources/kafka-src50
```


To gather logs

```
k -n knative-eventing logs -l=app=kafka-source-dispatcher --max-log-requests 10 -f | tee logs1a.txt
k -n knative-eventing logs -f kafka-source-controller- | tee -a kafka-source-controller.txt
k logs eventdisplay-9f4c7855-x42tp -f | tee logs2.txt
k logs duplicates1-kafkaproduce- | tee logs-kafkaproducer.txt
```

To monitor test exeuciton:

```
kubectl run curl --image=curlimages/curl --rm=true --restart=Never -ti -- http://kafkascraper/stats
source kafka-src50 received: 5000 messages, 3959 are dups and 0 non 200s.
```

```
kubectl -n kafka run kafka-consumer-groups -ti --image=quay.io/strimzi/kafka:0.28.0-kafka-3.1.0 --rm=true --restart=Never -- bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe  --group kafka-src50c
```

