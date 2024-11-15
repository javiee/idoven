
    Which database would you choose?
    How would you manage deployments?
    What approach would you take for backups?
    How would you implement monitoring?
    Identify any potential gray areas that could affect functionality and propose solutions.
    If we want to transition to an active-active scenario, what modifications would be necessary?


Assumptions

- Open source tools, non-propiotary 
- The system is under no heavy workload
- Providing the easiest solution with the least operation overhead

1 - For an entrepise solution I would choose to use Posgresql

Although the exercise does not make any reference about wether the application performs heavy writes or reads worlkloads to the database, the reason of choosing posgresql are its extended capabilities for clustering and replication, since Postgress is based on WAL files, making faster and  more reliable than mysql. Since we are looking for deploying our databases in multiple clouds I believe replication realibility will be paramount to maintain consistency across the stacks.

2 - To deploy a posgresql cluster in kubernetes I would use a helm chart , probably this one 
https://artifacthub.io/packages/helm/bitnami/postgresql. Bitnami 

For this scenario:

Deploying two Bitnami PostgreSQL clusters:

* Cluster 1 (Active): Primary node and a local secondary replica.

* Cluster 2 (Standby): Primary node and a local secondary replica, configured as a standby of Cluster 1.

Setting up asynchronous replication between Cluster 1 and Cluster 2, where Cluster 2’s primary will act as a read-only standby.

example config:

GCP Cluster:
```
helm install gcp-postgresql-primary bitnami/postgresql \
  --set global.postgresql.auth.postgresPassword=password \
  --set global.postgresql.auth.replicationPassword=passwordreplication \
  --set architecture=replication \
  --set auth.replicationUsername=replicator \
  --set auth.replicationPassword=replicationPassword \
  --set primary.service.type=LoadBalancer
```
Note that in order to make this work we need to expose the kubernetes service as Loadbalancer type, hence this config is needed.
```
--set primary.service.type=LoadBalancer
```
AWS Cluster:
```
helm install aws-postgresql-standby bitnami/postgresql \
  --set global.postgresql.auth.postgresPassword=password \
  --set global.postgresql.auth.replicationPassword=passwordreplicat ion \
  --set architecture=replication \
  --set auth.replicationUsername=replicator \
  --set auth.replicationPassword=replicationPassword \
  --set primary.standby.enabled=true
  --set primary.standby.primaryHost=gcp-postgresql-primary-dns \
  --set primary.standby.primaryPor=5432 \
  --set primary.service.type=LoadBalancer
```
To deploy helm charts into kubernetes I would consider 2 options flux2 or argocd. For this particular
scenario I would vote in favour or argocd due to its support for multitenancy deployments. 
You could deploy the above by using and application set

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cloud-posgresql-cluster
  namespace: argocd
spec:
  generators:
  - list:
      elements:
        - cluster: gcp
          server: https://gcp-cluster-url    
          values: |
            architecture: replication
            primary
              standby
                enabled: false
          ...

        - cluster: aws
          server: https://aws-cluster-url  
          values: |
            architecture: replication
            primary
              standby
                enabled: true   
            ...
  template:
    metadata:
      name: "{{cluster}}-postgresql-cluster"           # Name will be dynamically set per cluster
    spec:
      project: default
      source:
        repoURL: "bitnami/posgresql"  # URL of your Helm chart repository
        chart: ""        # Name of the Helm chart
        targetRevision: "<chart-version>"   # Version of the Helm chart
      helm:
          values: "{{values}}"    
      destination:
        server: "{{server}}"         # Server URL will be dynamically set per cluster
        namespace: default           # Target namespace for the application
      syncPolicy:
        automated:                   # Optional: Enable auto-sync
          prune: true
          selfHeal: true

3 - What approach would you take for backups?
Assuming choosing a solution for scalable backups I would probably use Velero

https://github.com/vmware-tanzu/velero

The advantages of using this approach are the following:

* Open source project with a lot of starts, contributions and recently updated.
* It supports on-prem as well as multiple clouds (Azure, GCP and AWS).
* Scheduled backups.
* Options. Snapshots or filesystem backups
* It is not tied to a specific storage platform.
* It can be easily restore pvc into another cluster, for example restoring prod database into staging environment for troubleshooting and issue
* One tool for all.

4 - 
Assumtions: 
- Prometheus operator is installed inside the kubernetes cluster

https://github.com/prometheus-community/postgres_exporter

5 - This set up got multiple problems that can be addressed 

* heavy / writes workloads. This setup only allows one primary that can accepts writes. This mean that writes only allows scale vertically the machine so it there is a limit to where we can reach.  We could potentially address this problem by using sharding.

* Kubernetes installation primary/secondary does not support failover. That means in the event of master going off-line we will loose the ability of accepting writes until an operator fails over the cluster in favour of one the replicas ones. Depending of the workload this could be tacke of different 2 ways in my opinion.
Easy-way. Since we will have monitoring in place, we can detect when one the primaries is down for some time and then trigger a k8 job or script to automatically failover. I think this scenario is ok if we have certain tolerance for writes queries to fail for some time.

Hard-way. If quick failover is needed in order to support the business or the cluster is under heavy workloads, then we need to change our approach. THen we need to consider to use prometheues operator uses Patroni under the hood to manage high availability, automated failover, and leader election. When a primary pod crashes, Patroni detects the failure and coordinates a failover to a healthy replica in the cluster. 

https://github.com/zalando/postgres-operator

Example:
GCP cluster
```
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: primary-db-cluster
  namespace: default
spec:
  teamId: "team-name"
  volume:
    size: "10Gi"
  numberOfInstances: 2
  users:
    app_user:
      - superuser
      - createdb
  databases:
    app_db: app_user
  postgresql:
    version: "14"
  standby:
    enabled: false
  enableMasterLoadBalancer: true    # Enables external access to the primary service
  enableReplicaLoadBalancer: true   # Enables external access to the replicas
  exposeMasterService: true         # Exposes the master as a service
  exposeReplicaService: true        # Exposes replicas as services
  service:
    type: ClusterIP                 # Set Service type to ClusterIP for internal communication

```
AWS cluster (standby)
```
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: standby-db-cluster
  namespace: default
spec:
  volume:
    size: "10Gi"
  numberOfInstances: 2
  users:
    app_user:
      - superuser
      - createdb
  databases:
    app_db: app_user
  postgresql:
    version: "14"
  standby:
    enabled: true
    primary_conninfo: "host=<primary-cluster-loadbalancer-ip> port=5432 user=replicator password=<replication-password>"
  enableMasterLoadBalancer: true
  enableReplicaLoadBalancer: true
  exposeMasterService: true
  exposeReplicaService: true
  service:
    type: ClusterIP
```

This setup is far more complex and in my opinion should be avoided unless there are strong requirements that leads us to implement this operator. 

Failover would happen within the cluster so to fail over AWS stack we would need to do manually or by running an automation. 

6 - Transitioning to active / passive to active / active we need to configure the aplications of stack B to performs writes on the primary that is on the GCP cluster. The stanby cluster can still be use to serve read queries. In order to provide realibility to both cluster I would install connection pool like pgbouncer. By doing this we can ensure that connections are load balancer, we can take down db units without downtime to perform maintenance etc among other benefits. If the load would increase that one primary would be not enough my next move would be to consider sharding the database. 











https://www.pgbouncer.org/

In terms of monitoring prometheus.
