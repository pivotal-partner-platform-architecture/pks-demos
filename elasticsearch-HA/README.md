# pks-demos

## Elasticsearch with Persistent Volumes

Based on [Kubernetes samples](https://github.com/kubernetes/examples/tree/master/staging/elasticsearch) and Jaime Aguilar's Kubo demos (@jaimegag)

Demonstrates deployment of a single node Elasticsearch cluster with persistent volumes. Once deployed, we can demonstrate K8S self-healing, BOSH resurrection, and persistence capabilities.

## Prerequisites
1. A provisioned kubernetes cluster with at least 2 worker nodes. See PKS documentation to install PKS provision a k8s cluster, and connect via `kubectl`. https://docs.pivotal.io/runtimes/pks/1-0/

2. Please make sure that "Enable Privileged Containers - Use with caution" option is selected for your Plan (Plan 1, Plan 2 etc) under PKS Configuration in Ops Manager.

3. Login to PKS Environment `pks login -a <<PKS hostname>> -u <<username>> -p <<password>> --skip-ssl-verification`

4. Create a Kubernetes Cluster `pks create-cluster <<cluster-name>> --external-hostname <<Public DNS Name>> --plan <<plan-name>> --num-nodes <<no-of-nodes>>`

5. Login to the cluster `pks get-credentials <<cluster-name>>`

## Steps

1. Download this repository and execute the following commands.

2. Create Storage Class

`kubectl create -f storage-class-gcp.yml` OR
`kubectl create -f storage-class-vsphere`

3. Create Persistent Volume claim

`kubectl create -f persistent-volume-claim-es.yml`

4. Create Node Port Service

`kubectl create -f es-svc.yml`

This could also be changed to be a load balancer service if NSX-T is configured or running on GCP.

5. Create Elasticsearch Deployment

`kubectl create -f es-deployment.yml`


6. Access Elasticsearch Deployment

(The worker node ID is the next IP from your Kubernetes Master IP (x.x.x.17 if the Master IP is x.x.x.16), you can find the Kubernetes Master IP by executing `pks cluster <<cluster-name>>`)

`kubectl get svc` to get the node port. (Note down the 5 digit port number instead of 9200)
`export ES_IP=<<ES_WORKER_NODE_IP>>:<<node_port>>`
`curl http://$ES_IP` to validate elasticsearch is running.

7. (Optional) Inspect the Elasticsearch peristent volume file system.

(THIS STEP IS OPTIONAL, You can get the pod name from 'kubectl get pods' command, export pod name to $POD_NAME variable)

At this point you can run `kubectl exec $POD_NAME -it -- bash -il` to open a bash session in your elasticsearch container.
`cd /data` to inspect the persistent volume data structure before populating elastic search.

8. Populate Elasticsearch cluster with data

Add an index
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

9. Create a mapping of a type: order, which includes two properties - an ID and a customer_id.
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

10. Add two customer "documents" into the index with ids 1 and 2.
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

11. Validate we can request the data from Elasticsearch with curl:
`curl -i -X GET "http://$ES_IP/myindex/order/1?pretty=true"`

12. Testing PKS High Availability

Run this command to discover which worker node the elastic search pod is running on:
`kubectl get pods -o wide`
The information needed is available under Node, and correlates to the VM DNS Name in the IaaS (for this exercise, assuming vSphere).

Go to vCenter and "Power Off" the VM with the correlating VM DNS name.

Run `watch kubectl get pods -o wide` in your terminal to watch Kubernetes recreate the pod on the other worker. Note: This happens very quickly after VM is powered off!

Run `watch bosh -e <<your bosh alias>> vms` to watch BOSH recreate the missing worker. You can also use `watch kubectl get nodes -o wide`  in your terminal to see the new nodes being created by BOSH.

13. What just happened?

When we Powered Off the VM in vCenter, we not only lost the VM, we also lost the Elasticsearch pod which was running on that worker node. Two things happened:

I) Kubernetes replaced the Elasticsearch Pod:

When we saw the pod STATUS go from `Running`... to `Init`... to `Running`, we observed Kubernetes actions as it learned that the Elasticserach Pod was not available, and went on to initialize a new Elasticserach Pod. Kubernetes behaved this way because the Elasticsearch deployment utilized the Kubernetes concept of a [replicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/).

II) BOSH replaced the Worker Node:

When we saw the VM status go from  `running`... to `unresponsive agent` to `running`, we observed the BOSH actions as it learned that the Worker Node VM was no longer there, and went on to create a brand new VM in it's place. Since BOSH knows the role of that VM in the Kubernetes cluster, BOSH is able to recreate a brand new VM with that role, and our Kubernetes cluster is healthy again.

14. Validate we can still request the data from Elasticsearch with curl: curl -i -X GET "http://$ES_IP/myindex/order/1?pretty=true"

Since the Elasticsearch deployment is attached to a persistent volume, we maintain all the data entered even when the pod and VM are removed.
