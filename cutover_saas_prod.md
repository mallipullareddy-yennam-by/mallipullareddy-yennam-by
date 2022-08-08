
CutOver activites fron By side as DevOps
## Step19
## Purge topics on SaaS

- `kubectl config use-context gke_urban-saas-prod_us-east1-b_urban-prod-us-east1`
- kubectl exec -it debug-pod-0 -n urban-prod -- /bin/bash

- check current retention
`/opt/kafka/bin/kafka-topics.sh --zookeeper zookeeper-hs.urban-prod.svc.cluster.local:2181 --describe --topic urban-yas-reservation-order-backup-us-east1`
`/opt/kafka/bin/kafka-topics.sh --zookeeper zookeeper-hs.urban-prod.svc.cluster.local:2181 --describe --topic urban-yas-product-reservation-demand-backup-us-east1`

- check messsages in topics if they're no messges no need purge topis if any messages go to next step and purge 
`/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-hs.urban-prod.svc.cluster.local:9092 --topic urban-yas-reservation-order-backup-us-east1 --from-beginning --property print.key=true`
`/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-hs.urban-prod.svc.cluster.local:9092 --topic urban-yas-product-reservation-demand-backup-us-east1 --from-beginning --property print.key=true`

- set retention to 1000
`/opt/kafka/bin/kafka-configs.sh --zookeeper zookeeper-hs.urban-prod.svc.cluster.local:2181 --alter --entity-name urban-yas-reservation-order-backup-us-east1 --entity-type topics --add-config retention.ms=1000`
`/opt/kafka/bin/kafka-configs.sh --zookeeper zookeeper-hs.urban-prod.svc.cluster.local:2181 --alter --entity-name urban-yas-product-reservation-demand-backup-us-east1 --entity-type topics --add-config retention.ms=1000`

- wait for meesage to clear
`/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-hs.urban-prod.svc.cluster.local:9092 --topic urban-yas-reservation-order-backup-us-east1 --from-beginning --property print.key=true`
`/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-hs.urban-prod.svc.cluster.local:9092 --topic urban-yas-product-reservation-demand-backup-us-east1 --from-beginning --property print.key=true`

- revert retention to default 
`/opt/kafka/bin/kafka-configs.sh --zookeeper zookeeper-hs.urban-prod.svc.cluster.local:2181 --alter --entity-name urban-yas-reservation-order-backup-us-east1 --entity-type topics --delete-config retention.ms`
`/opt/kafka/bin/kafka-configs.sh --zookeeper zookeeper-hs.urban-prod.svc.cluster.local:2181 --alter --entity-name urban-yas-product-reservation-demand-backup-us-east1 --entity-type topics --delete-config retention.ms`

- check retention period
`/opt/kafka/bin/kafka-topics.sh --zookeeper zookeeper-hs.urban-prod.svc.cluster.local:2181 --describe --topic urban-yas-reservation-order-backup-us-east1`
`/opt/kafka/bin/kafka-topics.sh --zookeeper zookeeper-hs.urban-prod.svc.cluster.local:2181 --describe --topic urban-yas-product-reservation-demand-backup-us-east1`

## Step35
- use gsutill to download the file extract from on-prem
```gsutil cp gs://<bucket_name>/<extract_file_name>```

## Step36
- import Capacity tables to saas, restart capacity services
Tables :
- Truncate tables 
`TRUNCATE TABLE location_ft_day_capacity;`
`TRUNCATE TABLE loc_day_capacity;`
`TRUNCATE TABLE location_ft_capacity;`
`TRUNCATE TABLE order_capacity;`
`TRUNCATE TABLE location_capacity;`
`TRUNCATE TABLE order_capacity_waiting_confirm;`
`TRUNCATE TABLE publish_schedule;`

- Import tables
******PREPARE SCRIPT AND TEST IT LOCAL******

Update the dump loation in below comamnds 
`./dsbulk-1.8.0/bin/dsbulk load -h 10.13.96.5 -t 9042 -u cassandra -p cassandra -c json --connector.csv.maxCharsPerColumn 40960 --batch.mode DISABLED  -k urban_prod -t location_ft_day_capacity -url /sstables/data/backups/`
`./dsbulk-1.8.0/bin/dsbulk load -h 10.13.96.5 -t 9042 -u cassandra -p cassandra -c json --connector.csv.maxCharsPerColumn 40960 --batch.mode DISABLED  -k urban_prod -t loc_day_capacity -url /sstables/data/backups/`
`./dsbulk-1.8.0/bin/dsbulk load -h 10.13.96.5 -t 9042 -u cassandra -p cassandra -c json --connector.csv.maxCharsPerColumn 40960 --batch.mode DISABLED  -k urban_prod -t location_ft_capacity -url /sstables/data/backups/`
`./dsbulk-1.8.0/bin/dsbulk load -h 10.13.96.5 -t 9042 -u cassandra -p cassandra -c json --connector.csv.maxCharsPerColumn 40960 --batch.mode DISABLED  -k urban_prod -t order_capacity -url /sstables/data/backups/`
`./dsbulk-1.8.0/bin/dsbulk load -h 10.13.96.5 -t 9042 -u cassandra -p cassandra -c json --connector.csv.maxCharsPerColumn 40960 --batch.mode DISABLED  -k urban_prod -t location_capacity -url /sstables/data/backups/`
`./dsbulk-1.8.0/bin/dsbulk load -h 10.13.96.5 -t 9042 -u cassandra -p cassandra -c json --connector.csv.maxCharsPerColumn 40960 --batch.mode DISABLED  -k urban_prod -t order_capacity_waiting_confirm -url /sstables/data/backups/`


- Restart capacity

Stop
`kubectl scale statefulsets capacity-app --replicas=0 -n urban-prod`
`kubectl scale deployment capacity-streamer --replicas=0 -n urban-prod`
`kubectl scale statefulsets capacity-hazelcast-scheduler --replicas=0 -n urban-prod`
`kubectl scale statefulsets capacity-hazelcast --replicas=0 -n urban-prod`
`kubectl scale deployment capacity-change-events-consumer-app --replicas=0 -n urban-prod`

Start
`kubectl scale statefulsets capacity-hazelcast --replicas=3 -n urban-prod`
`kubectl scale statefulsets capacity-hazelcast-scheduler --replicas=1 -n urban-prod`
`kubectl scale statefulsets capacity-app --replicas=3 -n urban-prod`
`kubectl scale deployment capacity-streamer --replicas=3 -n urban-prod`
`kubectl scale deployment capacity-change-events-consumer-app --replicas=0 -n urban-prod`

## Step40 (Maybe On-Prem)

- Wait for topics URBAN-impl-warehouse-supply-US lag to be consumed by urban-shipment-update-streamer
    	
`/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group common-master-data --describe | grep URBAN-impl-warehouse-supply-US`

- Wait for topics URBAN-impl-warehouse-supply-US lag to be consumed by urban-warehouse-supply-streamer
`/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group common-master-data-saas --describe | grep URBAN-impl-warehouse-supply-US`

## Step46
- Wait for lag to be consumed for availability-migration-consumer-app - On SaaS

```/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server kafka-hs.urban-qa.svc.cluster.local:9092 --group common-services-updates-streamer --describe| grep 'urban-yas-reservation-order-migration-integration-topic-us-east1'```

```/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server kafka-hs.urban-prod.svc.cluster.local:9092 --group common-services-updates-streamer --describe| grep 'urban-yas-product-reservation-demand-migration-integration-topic-us-east1'```

## Step50
- Stop Demand Consumer On-SaaS scale to 0

[availability-migration-consumer-app]
(https://console.cloud.google.com/kubernetes/statefulset/us-east1-b/urban-prod-us-east1/urban-prod/availability-migration-consumer-app/details?project=urban-saas-prod)

## Step56
- validate topics on SaaS is empty 

```/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server kafka-hs.urban-prod.svc.cluster.local:9092 --group common-services-updates-streamer --describe| grep 'urban-yas-reservation-order-migration-integration-topic-us-east1'```

```/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server kafka-hs.urban-prod.svc.cluster.local:9092 --group common-services-updates-streamer --describe| grep 'urban-yas-product-reservation-demand-migration-integration-topic-us-east1'```

- check messsages in topics if they're no messges no need purge topis if any messages go to next step and purge 
`/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-hs.urban-prod.svc.cluster.local:9092 --topic urban-yas-reservation-order-migration-integration-topic-us-east1 --from-beginning --property print.key=true`
`/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-hs.urban-prod.svc.cluster.local:9092 --topic urban-yas-product-reservation-demand-migration-integration-topic-us-east1 --from-beginning --property print.key=true`


## Step57
- Start SaaS Demand ReplicatorScale to 1 pods 
- [availability-changes-to-backup-app](https://console.cloud.google.com/kubernetes/statefulset/us-east1-b/urban-prod-us-east1/urban-prod/availability-changes-to-backup-app?project=urban-saas-prod)
- [availability-backup-to-migration-app](https://console.cloud.google.com/kubernetes/statefulset/us-east1-b/urban-prod-us-east1/urban-prod/availability-backup-to-migration-app/details?project=urban-saas-prod)

## Step 66
- Validate ATP Publish Consumers are running on SaaS
- 2 pods [availability-location-atp-publish-consumer](https://console.cloud.google.com/kubernetes/deployment/us-east1-b/urban-prod-us-east1/urban-prod/availability-location-atp-publish-consumer/overview?project=urban-saas-prod)
- 12 pods [availability-network-atp-publish-consumer](https://console.cloud.google.com/kubernetes/deployment/us-east1-b/urban-prod-us-east1/urban-prod/availability-network-atp-publish-consumer/overview?project=urban-saas-prod)

## Step 85 
- Trigger demand healer manually at 1am EST and validate the number of records getting updated are similar to onprem numbers, if it looks good redeploy with cron enabled for 1AM & 5AM EST runs - in SaaS yas-demand-mismatch-app

- scale [yas-demand-mismatch-app](https://console.cloud.google.com/kubernetes/statefulset/us-east1-b/urban-prod-us-east1/urban-prod/yas-demand-mismatch-app/details?project=urban-saas-prod)
