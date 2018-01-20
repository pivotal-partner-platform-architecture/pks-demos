# pks-demos

## Elasticsearch with Persistent Volumes

Based on [Kubernetes samples](https://github.com/kubernetes/examples/tree/master/staging/elasticsearch) and Jaime Aguilar's Kubo demos (@jaimegag)

Demonstrates deployment of a single node Elasticsearch cluster with persistent volumes. Once deployed, we can demonstrate K8S self-healing, BOSH resurrection, and persistence capabilities.

1. Create Storage Class

`kubectl create -f storage-class-gcp.yml` OR
`kubectl create -f storage-class-vsphere`

2. Create Persistent Volume claim

`kubectl create -f persistent-volume-claim-es.yml`

3. Create Load Balancer Service

`kubectl create -f es-svc.yml`

4. Create Elastic Search Deployment

`kubectl create -f es-deployment.yml`
`kubectl get svc` to get the exernal IP address
`export ES_IP=<<ES_External_IP>>`
`curl http://$ES_IP:9200` to validate elasticsearch is running

5. Populate Elasticsearch cluster with data

At this point you can run `kubectl exec $POD_NAME -it -- bash -il` to open a bash session in your elasticsearch container.
`cd /data` to inspect the persistent volume data structure before populating elastic search.

Add an index
```
curl -X PUT http://$ES_IP:9200/myindex -d \
'{"settings":
  {
    "index": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    }
  }
}'
```

`curl -X GET "http://$ES_IP:9200/myindex/_settings?pretty=true"`

Then we are going to create a mapping of a type: order, which includes two properties - an ID and a customer_id.
```
curl -X POST http://$ES_IP:9200/myindex/order/_mapping -d \
'{"order":
  {
    "properties": {
      "id": {"type": "keyword", "store": "yes"},
      "customer_id": {"type": "keyword", "store": "yes"}
    }
  }
}'
```
With this, we add two customer "documents" into the index with ids 1 and 2.
```
curl -i -H "Content-Type: application/json" -X POST "http://$ES_IP:9200/myindex/order/1?pretty=true" -d \
'{
  "customer_id": "customer1"
}'

curl -i -H "Content-Type: application/json" -X POST "http://$ES_IP:9200/myindex/order/2?pretty=true" -d \
'{
  "customer_id": "customer2"
}'
```

To validate we can request the data from elastic search:
`curl -i -X GET "http://$ES_IP:9200/myindex/order/1?pretty=true"`

6. Testing PKS HA
