Title: Elasticsearch Cluster on Google Kubernetes Engine
Date: 2020-09-07 17:30
Modified: 2020-09-07 17:30
Category: DevOps
Tags: Elasticsearch, Kubernetes
Slug: Elasticsearch-Cluster-on-Google-Kubernetes-Engine
Authors: Kishore Chandran
Summary: My experience on creating an Elasticsearch Cluster on Google Kubernetes Engine

# Elasticsearch Cluster on Google Kubernetes Engine
From being an Java Developer this is the first time I am wetting my hands on Elasticsearch and Kubernetes, 
though I hear a lot about them never really had a chance to work on those. 
Here I share my experience on creating an Elasticsearch Cluster on Google Kubernetes Engine.

These are the following steps involved in the process,

1. Provision the required clusters on [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine).
2. Enable the persistent volume via [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/).
3. Enable the Elasticsearch node discovery via [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services).
4. Deploy the Elasticsearch Cluster via [Stateful Sets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/).

## Provisioning the Cluster

