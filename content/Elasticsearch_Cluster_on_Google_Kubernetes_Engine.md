Title: Elasticsearch Cluster on Google Kubernetes Engine
Date: 2020-09-07 17:30
Modified: 2020-09-07 17:30
Category: DevOps
Tags: Elasticsearch, Kubernetes
Slug: Elasticsearch-Cluster-on-Google-Kubernetes-Engine
Authors: Kishore Chandran
Summary: My experience on creating an Elasticsearch Cluster on Google Kubernetes Engine


From being an Java Developer this is the first time I am wetting my hands on Elasticsearch and Kubernetes, 
though I hear a lot about them never really had a chance to work on those. 
Here I share my experience on creating an Elasticsearch Cluster on Google Kubernetes Engine.

These are the following steps involved in the process,

1. Provision the required clusters on [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine).
2. Enable Persistent volume using [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/).
3. Enable Elasticsearch node discovery using [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services).
4. Deploy Elasticsearch Cluster using [Stateful Sets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/).

## Provisioning the Cluster
Let us first provision the clusters in Google Kubernetes Engine for our Elasticsearch. We are going to create 3 node pools as below,

1. master-nodes - 3 nodes with 2vCPU, 8GB Memory and Standard persistent disk of 10GB as default
2. data-nodes - 5 nodes with 16vCPU, 64GB Memory and Standard persistent disk of 10GB as default
3. coordinator-only - 5 nodes with 4vCPU, 8GB Memory and Standard persistent disk of 10GB as default

You can use the below command to provision.
```
gcloud beta container --project "my-project" clusters create "my-cluster" --zone "us-central1-c" \
--no-enable-basic-auth --cluster-version "1.15.12-gke.2" --machine-type "e2-standard-2" --image-type "COS" \
--disk-type "pd-standard" --disk-size "10" --metadata disable-legacy-endpoints=true \
--scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
--num-nodes "3" --enable-stackdriver-kubernetes --enable-ip-alias --network "projects/datamastering/global/networks/default" \
--subnetwork "projects/datamastering/regions/us-central1/subnetworks/default" --default-max-pods-per-node "110" \
--no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver \
--enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 && gcloud beta container \
--project "datamastering" node-pools create "data-nodes" \
--cluster "my-cluster" --zone "us-central1-c" --node-version "1.15.12-gke.2" --machine-type "e2-standard-16" --image-type "COS" \
--disk-type "pd-standard" --disk-size "10" --metadata disable-legacy-endpoints=true \
--scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
--num-nodes "5" --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 && gcloud beta container \
--project "datamastering" node-pools create "coordinator-only" --cluster "my-cluster" --zone "us-central1-c" \
--node-version "1.15.12-gke.2" --machine-type "e2-custom-4-8192" --image-type "COS" --disk-type "pd-standard" \
--disk-size "10" --metadata disable-legacy-endpoints=true \
--scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
--num-nodes "2" --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0
```
> **_NOTE:_**  In case you are provisioning the cluster in console make sure to enable `Enable Computer Engine Persistent Disk CSI Driver` 
> under Features.

## Enable Persistent Volume
Next we create the storage classes for our Elasticsearch cluster, Create new file called `storage.yaml` with the following content:

```
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: pd-ssd
parameters:
  type: pd-ssd
provisioner: pd.csi.storage.gke.io
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

Here we use `pd.csi.storage.gke.io` as our provisioner, this is the reason we enabled 
`Enable Computer Engine Persistent Disk CSI Driver` under Features of cluster creation.

In case you dont want to use the above provisioner you can also use the `kubernetes.io/gce-pd` as bellow.

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: pd-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  zone: us-central1-c
```
  
  > **_NOTE:_**  Use only one of the above since we have used the same name for both, in case you want to use both the 
>configurations use different names for `pd-ssd`.

Now to apply the changes to our cluster first get credentials of our cluster using the bellow command.

```gcloud container clusters get-credentials my-cluster --zone us-central1-c --project my-project```

To apply our changes use the below command.

```kubectl apply -f storage.yaml```

## Elasticsearch Node Discovery
Create new file called `service.yaml` with the following content.

```
apiVersion: v1
kind: Service
metadata:
  labels:
  name: elasticsearch-k8s-es-lb
  namespace: default
  annotations:
    # NOTE: We recommend only exposing ES on your internal network
    cloud.google.com/load-balancer-type: "Internal"
    # If you are using external DNS you can have it publish the dns records
    # for this service for you
    external-dns.alpha.kubernetes.io/hostname: "es-lb.example.com."
    external-dns.alpha.kubernetes.io/ttl: "30"
spec:
  type: LoadBalancer
  ports:
  - name: https
    port: 9200
    protocol: TCP
    targetPort: 9200
  selector:
    common.k8s.elastic.co/type: elasticsearch
    elasticsearch.k8s.elastic.co/cluster-name: elasticsearch-k8s
  sessionAffinity: None
```

Then enable the headless service using the following command

```kubectl apply -f service.yaml```

## Deploy Elasticsearch Cluster
The last step is to deploy the elasticsearch cluster, to deploy the ECK operator run,

```kubectl apply -f https://download.elastic.co/downloads/eck/1.2.0/all-in-one.yaml```
This will deploy it into the elastic-system namespace. For latest version of the operator you find install instructions 
[here](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html).

To deploy to our cluster create `elasticsearch.yaml` with the below content.

```
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-k8s
spec:
  version: 6.8.3
  nodeSets:
  - name: masters
    count: 3
    podTemplate:
      spec:
        containers:
        - name: master-nodes
          image: docker.elastic.co/elasticsearch/elasticsearch:6.8.3
          env:
          - name: ES_JAVA_OPTS
            value: -Xms6g -Xmx6g
          - name: cluster.name
            value: my-cluster
          - name: node.name
            value: master-nodes
          resources:
            requests:
              memory: 8Gi
              cpu: 2
            limits:
              memory: 8Gi
              cpu: 2
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        # see https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-init-containers-plugin-downloads.html
        - name: install-plugins
          command:
          - sh
          - -c
          - |
            bin/elasticsearch-plugin install --batch repository-gcs
    config:
      node.master: true
      node.data: false
      node.ingest: false
      script.allowed_types: inline
      path.repo: /
      indices.fielddata.cache.size: 20%
      indices.query.bool.max_clause_count: 4096
      indices.memory.index_buffer_size: "25%"
    volumeClaimTemplates:
    - metadata:
        name: masters-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 500Gi
        storageClassName: pd-ssd
  - name: data-nodes
    count: 5
    podTemplate:
      spec:
        containers:
        - name: data-nodes
          image: docker.elastic.co/elasticsearch/elasticsearch:6.8.3
          env:
          - name: ES_JAVA_OPTS
            value: -Xms28g -Xmx28g
          - name: cluster.name
            value: my-cluster
          - name: node.name
            value: data-nodes
          resources:
            requests:
              memory: 36Gi
              cpu: 4
            limits:
              memory: 36Gi
              cpu: 4
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        # see https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-init-containers-plugin-downloads.html
        - name: install-plugins
          command:
          - sh
          - -c
          - |
            bin/elasticsearch-plugin install --batch repository-gcs
    config:
      node.master: false
      node.data: true
      script.allowed_types: inline
      path.repo: /
      indices.fielddata.cache.size: 20%
      indices.query.bool.max_clause_count: 4096
      indices.memory.index_buffer_size: "25%"
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 500Gi
        storageClassName: pd-ssd
  - name: coordinator-only
    count: 2
    podTemplate:
      spec:
        containers:
        - name: coordinator-only
          image: docker.elastic.co/elasticsearch/elasticsearch:6.8.3
          env:
          - name: ES_JAVA_OPTS
            value: -Xms4g -Xmx4g
          - name: cluster.name
            value: my-cluster
          - name: node.name
            value: coordinator-only
          resources:
            requests:
              memory: 6Gi
              cpu: 2
            limits:
              memory: 6Gi
              cpu: 3
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        # see https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-init-containers-plugin-downloads.html
        - name: install-plugins
          command:
          - sh
          - -c
          - |
            bin/elasticsearch-plugin install --batch repository-gcs
    config:
      node.master: false
      node.data: false
      node.ingest: true
      script.allowed_types: inline
      path.repo: /
      indices.fielddata.cache.size: 20%
      indices.query.bool.max_clause_count: 4096
      indices.memory.index_buffer_size: "25%"
    volumeClaimTemplates:
    - metadata:
        name: coordinator-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 500Gi
        storageClassName: pd-ssd
  http:
    service:
      spec:
        type: ClusterIP
    # see https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-custom-http-certificate.html
    # tls:
    #   certificate:
    #     secretName: "example.com"

```
To apply the yaml use the bellow command
```kubectl apply -f elasticsearch.yaml```

Now your cluster must be up, confirm the same in Workloads section in the console or you can use the bellow command and see
if all the pods are in running state.
```kubectl get pods```

## Test Your Cluster
To verify that your cluster is working fine use the below commands to access the content of the cluster

```
# get the generated password
PASSWORD=$(kubectl get secret elasticsearch-k8s-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')

# using the load balancer we created
ENDPOINT=$(kubectl get svc elasticsearch-k8s-es-lb -o go-template='{{ (index .status.loadBalancer.ingress 0).ip }}')
curl -u "elastic:$PASSWORD" -k https://$ENDPOINT:9200

# To test locally can set up proxy
kubectl port-forward service/elasticsearch-k8s-es-http 9200
curl -u "elastic:$PASSWORD" -k "https://localhost:9200"
```
