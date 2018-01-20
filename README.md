# pks-demos

Elasticsearch with Persistent Volumes

Demonstrate deployment of a single node Elasticsearch with persistent volumes. Once deployed, we can demonstrate K8S self-healing, BOSH resurrection, and persistence capabilities.

Based on Kubernetes samples (https://github.com/kubernetes/examples/tree/master/staging/elasticsearch) and Jaime Aguilar (@jaimegag)

1. Create Storage Class

2. Create Persistent Volume claim

3. Create Load Balancer Service

4. Create Elastic Search Deployment

5. Populate Elasticsearch cluster with data
