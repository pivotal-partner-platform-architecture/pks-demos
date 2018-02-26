# pks-demos

## Elasticsearch with Persistent Volumes

Based on [Kubernetes samples](https://github.com/kubernetes/examples/tree/master/staging/elasticsearch) and Jaime Aguilar's Kubo demos (@jaimegag)

Demonstrates deployment of a single node Elasticsearch cluster with persistent volumes. Once deployed, we can demonstrate K8S self-healing, BOSH resurrection, and persistence capabilities.

## Prerequisites
1. A provisioned kubernetes cluster with at least 2 worker nodes. See PKS documentation to install PKS provision a k8s cluster, and connect via `kubectl`. https://docs.pivotal.io/runtimes/pks/1-0/

2. Please make sure that "Enable Privileged Containers - Use with caution" option is selected for your Plan (Plan 1, Plan 2 etc) under PKS Configuration in Ops Manager.

## Steps

1. Clone this repository.

2. Create Storage Class

`kubectl create -f storage-class-gcp.yml` OR
`kubectl create -f storage-class-vsphere`

3. Create Persistent Volume claim

`kubectl create -f persistent-volume-claim-es.yml`

4. Create Node Port Service

`kubectl create -f es-svc.yml`

This could also be changed to be a load balancer service if NSX-T is configured or running on GCP.

5. Create Elastic Search Deployment (The worker node ID is the next IP  from your Kubernetes Master IP (x.x.x.17 if the Master IP is x.x.x.16), you can find the Kubernetes Master IP by executing  'PKS cluster <<cluster-name>>')

`kubectl create -f es-deployment.yml`
`kubectl get svc` to get the port.
`export ES_IP=<<ES_WORKER_NODE_IP>>:<<node_port>>` (Please use the 5 digit port number instead of 9200)
`curl http://$ES_IP` to validate elasticsearch is running.

6. Populate Elasticsearch cluster with data (THIS STEP IS OPTIONAL , You can get the pod name from 'kubectl get svc' command, export pod name to $POD_NAME variable)

At this point you can run `kubectl exec $POD_NAME -it -- bash -il` to open a bash session in your elasticsearch container.
`cd /data` to inspect the persistent volume data structure before populating elastic search.

7. Add an index
```
curl -H'Content-Type: application/json' -X PUT http://$ES_IP/myindex -d '
{"settings":
  {
    "index": {
      "number_of_shards": 2,
      "number_of_replicas": 1
    }
  }
}'
```

`curl -X GET "http://$ES_IP/myindex/_settings?pretty=true"`

8. Then we are going to create a mapping of a type: order, which includes two properties - an ID and a customer_id.
```
curl -H'Content-Type: application/json' -X POST http://$ES_IP/myindex/order/_mapping -d \ '
{"order":
  {
    "properties": {
      "id": {"type": "keyword", "store": "true"},
      "customer_id": {"type": "keyword", "store": "true"}
    }
  }
}'
```

9. With this, we add two customer "documents" into the index with ids 1 and 2.
```
curl -i -H "Content-Type: application/json" -X POST "http://$ES_IP/myindex/order/1?pretty=true" -d \
'{
  "customer_id": "customer1"
}'

curl -i -H "Content-Type: application/json" -X POST "http://$ES_IP/myindex/order/2?pretty=true" -d \
'{
  "customer_id": "customer2"
}'
```

10. To validate we can request the data from elastic search:
`curl -i -X GET "http://$ES_IP/myindex/order/1?pretty=true"`

11. Testing PKS HA

Run this command to discover which worker node the elastic search pod is running on:
`kubectl get pods -o wide` 
This informtion is available under Node. You may also use 'kubectl get nodes -o wide' and fetch the node that corresponds to the Worker node IP you fetched in step 5)

Go to vCenter and "Power Off" the VM.

Run `watch kubectl get pods -o wide` to watch Kubernetes recreate the pod on the other worker.

Run `watch bosh -e <<your bosh alias>> vms` to watch BOSH recreate the missing worker.
