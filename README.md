# pks-demos

Elasticsearch with Persistent Volumes

Demonstrate deployment of a single node Elasticsearch with persistent volumes. Once deployed, we can demonstrate K8S self-healing, BOSH resurrection, and persistence capabilities.

Based on [Kubernetes samples](https://github.com/kubernetes/examples/tree/master/staging/elasticsearch) and Jaime Aguilar's Kubo demos (@jaimegag)

1. Create Storage Class

`kubectl create -f storage-class-gcp.yml` OR
`kubectl create -f storage-class-vsphere`

2. Create Persistent Volume claim

`kubectl create -f persistent-volume-claim-es.yml`

3. Create Load Balancer Service

`kubectl create -f es-svc.yml`

4. Create Elastic Search Deployment

`kubectl create -f es-deployment.yml`

5. Populate Elasticsearch cluster with data
