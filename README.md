#   Demo: application portability with Kubernetes

##  1. Environment Preparation

### 1.1. Overview

This section details setup of the demo environment based on Kublr using
Kublr command line tools.

It includes two Kublr managed Kubernetes clusters - one in AWS and
another one in Azure.

Generally speaking, any other tool may be used to setup these Kubernetes
clusters (although it was not tested).

If tools other than Kublr are used to setup the clusters, the clusters
must satisfy requirements summarized in the section **1.2. Non-Kublr
created clusters**

### 1.3. Setting up clusters with Kublr

Before you get stated install Kublr-in-a-Box using instructions provided
in Kublr-in-a-Box Installation Guide: https://docs.kublr.com/installationguide/bootstrap/

1.  Set up two clusters, one in AWS and one in Azure, using the following guidelines:

    1.  Find more information on how to setup clusters in Kublr using Kublr Quick Start Guide:

        a.  AWS: https://docs.kublr.com/installationguide/production-control-plane/

        b.  Azure: https://docs.kublr.com/installationguide/production-control-plane-azure/

    2.  Each cluster includes 1 master and 3 worker nodes:

        a.  AWS: t2.medium instance type for both: masters and work nodes

        b.  Azure: Standard_D3_v2 instance type for both: masters and work nodes.

    3.  For both clusters turn on Ingress and configure Let's Encrypt

2.  Wait until clusters are in Running state.

3.  Setup domains for these two clusters by pointing wildcard DNS records
    to Load Balancers of the newly created clusters.

    More details can be found at [How to Configure DNS for a Cluster with Ingress
    Controller](https://docs.kublr.com/dns-setup/)

### 1.3. Non-Kublr created clusters

Two clusters are to be setup, one in AWS and the other in Azure.

1.  Each cluster includes at least 3 worker nodes.

2.  AWS Kubernetes cluster has AWS cloud controller enabled, and

    the cloud controller can operate EBS disks, and

    a StorageClass object `default` exists and is setup to dynamically
    provision AWS EBS backed persistent volumes.

3.  Azure cluster has Azure cloud controller enabled, and

    the cloud controller can operate Azure disks, and

    a StorageClass object `default` exists and is setup to dynamically
    provision Azure disk backed persistent volumes.

4.  Ceph and RBD file systems are properly supported on the worker nodes

    This is only required for Rook/Ceph related scenarios

5.  An ingress controller is deployed into each cluster;

    the ingress controllers should support SSL temination and
    integration with     Letsencrypt service (e.g.
    https://github.com/jetstack/kube-lego or
    https://github.com/jetstack/cert-manager/ ) for all demo scenarios
    to work correctly.

6.  Wildcard DNS records are configured for the ingress controllers:
    `*.port-aws.demo.kublr.com` and `*.port-azure.demo.kublr.com` for
    AWS and Azure correspondingly.

    Different domain names may be used, in which case some changes need
    to be made in the demo scenario and some value files:
    `values-*-host*.yaml`

    See section **1.5. Variations** for more details.

### 1.4. Prepare client environment

1.  Prepare Kubernetes config file to access the clusters

    1.  Download Kubernetes config files for your clusters into files
        `cluster-aws/out/config-portability-aws.yaml` and
        `cluster-azure/out/config-portability-azure.yaml`.

    2.  Merge files `cluster-aws/out/config-portability-aws.yaml` and
        `cluster-azure/out/config-portability-azure.yaml` into a single
        kubernetes config file `clusters/config-portability.yaml` by
        copying contents of sections `clusters`, `users` and `contexts`.

    3.  Make sure that contexts in the file are renamed to `aws` and `azure`
        for AWS and Azure clusters correspondingly, and `currentContext`
        property is not defined.

2.  Install the config file, `kubectl`, and `helm` commands as described in
    Kublr Quick Start Guide referenced above or in Kubernetes and Helm
    documentation respectively.

    All commands in this file assume that `KUBECONFIG` environment variable
    is set and exported as follows:

    ```
    export KUBECONFIG="$(pwd)/clusters/config-portability.yaml"
    ```

    Test `kubectl` and `helm` commands:

    ```
    kubectl --context=aws get nodes
    kubectl --context=azure get nodes
    helm --kube-context=aws list
    helm --kube-context=azure list
    ```

3.  Verify that cluster' load balancers are available:

    Test AWS load balancer: `http://<aws-elb-specific-address>.elb.amazonaws.com`
    (this address will be unique for your specific cluster).

    Test Azure load balancer: `http://<azure-lb-specific-ip>` (this address will be
    unique for your specific cluster).

    Both should open default 404 page.

### 1.5. Environmental Variations

When setting up a specific environment, some of the parameters may or
will vary:

-   Kublr private repository access credentials; see section
    **1.3.1. Configure access creadentials** for more details.

-   Ingress controllers endpoints will be different for different
    clusters.

-   Different domain names may be used.

    This document assumes that `*.port-aws.demo.kublr.com` and
    `*.port-azure.demo.kublr.com` DNS records are configured for AWS and
    Azure cluster ingress controllers correspondingly.

    Should different domain names are selected, corresponding changes
    need to be made in all `values-*-host*.yaml` files.

## 2. Demo use-cases

### 2.1. Ingress

1.  Deploy evaluation on AWS

    ```
    helm --kube-context=aws upgrade -i demo demo-wordpress -f values-evaluation.yaml
    ```

    Check that wordpress is deployed (in k8s dashboard) and start port
    forwarding.

    ```
    kubectl --context aws port-forward \
        $(kubectl --context aws get pods -l app=demo-demo-wordpress-wordpress -o custom-columns=name:metadata.name --no-headers=true) \
        8080:80
    ```

    Wait until all pods are deployed, running, and healthy (1-2 min).

    Open http://localhost:8080/

2.  Deploy with Ingress and HTTP.

    ```
    helm --kube-context=aws upgrade -i demo demo-wordpress -f values-evaluation-ingress.yaml
    ```

    Open `http://<ingress-load-balancer-address>/` to test.

3.  Deploy with Ingress and HTTPS.

    ```
    helm --kube-context=aws upgrade -i demo demo-wordpress -f values-evaluation-ingress-ssl.yaml
    ```

    Open HTTP address of the cluster ingress load balancer, it should be redirected to HTTPS address.

4.  Deploy with Ingress, HTTPS, and host routing.

    ```
    helm --kube-context=aws upgrade -i demo demo-wordpress -f values-evaluation-ingress-ssl-host-aws.yaml
    ```

    Wait until all pods are deployed and healthy.

    Open http://wp.port-aws.demo.kublr.com/ to test.

    It should be redirected to https://wp.port-aws.demo.kublr.com/ with correct certificate

The same deployments may be repeated on Azure, you just need to change
the context name from `aws` to `azure`, and `*-aws.yaml` configuration
files to corresponding `...-azure.yaml` files.

1.  For example, you can deploy evaluation version with ingress on Azure as
    follows

    ```
    helm --kube-context=azure upgrade -i demo demo-wordpress -f values-evaluation-ingress-ssl-host-azure.yaml
    ```

### 2.2. Managed persistence

1.  Deploy AWS with EBS persistence

    ```
    helm --kube-context=aws upgrade -i demo demo-wordpress -f values-persistent-ingress-ssl-host-aws.yaml
    ```

    After this wordpress will stop working, because mysql will have been
    restarted with new storage and Wordpress-initialized database will
    have been lost because of that.

    To fix this just restart wordpress pod so that wordpress can re-initialize
    its database on the new storage.

    ```
    kubectl --context aws delete pods -l app=demo-demo-wordpress-wordpress --now
    ```

    After application is initialized and started, it should be able to
    survive both wordpress and mysql pod restart.

### 2.3. Self-hosted persistence

To deploy Azure with Ceph persistence you need to first make sure that
Ceph cluster is deployed and `replicapool` replica pool is created in
that cluster.

#### 2.3.1. Deploy Rook operator for Ceph cluster

Make sure node labels are correct

```
for n in $(kubectl --context=azure get nodes -o=custom-columns=NAME:.metadata.name --no-headers); do
    echo -n "$n: "; kubectl --context=azure label --overwrite node "$n" "kubernetes.io/hostname=$n"
done
```

Deploy operator

```
kubectl --context=azure apply -f rook/rook-operator.yaml
```

It is not instantaneous, so you need to wait until operator is completely
started and available. For that check operator's pod using the following
command until you see that 1 agent pod for each node in the cluster and
1 operator pod in `Running` state:

```
# Check that it is deployed (1 operator, 1 agent per node should be running)
kubectl --context=azure get -n rook-system pods
```

Check and wait until you see all the required pods running.

Next deploy a Rook cluster and tools

```
# Rook cluster
kubectl --context=azure apply -f rook/rook-cluster.yaml
```

Make sure that all cluster components are running - you should be able to
see 1 api pod, 1 manager pod, 3 monitor pods, and 1 osd pod for each node
of the cluster (3 in our case):

```
# check the Rook cluster is deployed (1 api, 1 mgr, 3 mon, 1 osd per node)
kubectl --context=azure get -n rook pods
```

Check and wait until you see all the required pods running.

After cluster is started you can deploy a pod with Rook tools and check
Rook cluster status using them:

```
# tools
kubectl --context=azure apply -f rook/rook-tools.yaml

# test cluster and tools
kubectl --context=azure exec -n rook rook-tools -- rookctl status
```

You should see a detailed output describing Rook/Ceph cluster status, which
is expected to be all OK.

Prepare Ceph replica pool and storage class

```
kubectl --context=azure apply -f rook/rook-storageclass.yaml

# check pools (replicapool)
kubectl --context=azure exec -n rook rook-tools -- ceph osd pool ls detail
```

#### 2.3.2. Deploy on Azure with self-hosted persistence

As long as Rook operator is deployed, and Ceph cluster, replica pool,
and storage class are created, the demo application with Ceph
persistence may be deployed.

```
helm --kube-context=azure upgrade -i demo demo-wordpress -f values-persistent-ingress-ssl-host-azure.yaml
```

## 3. Cleanup for another demo

Delete applications

```
helm --kube-context=aws delete --purge demo
helm --kube-context=azure delete --purge demo
```

Check that corresponding images are deleted from the replica pool

```
kubectl --context=azure exec -n rook rook-tools -- rbd list replicapool
# response should be empty
```

Delete storage class and replica pool

```
kubectl --context=azure delete -f rook/rook-storageclass.yaml
```

Check that corresponding pool is deleted

```
kubectl --context=azure exec -n rook rook-tools -- ceph osd pool ls
```

Delete rook cluster

```
kubectl --context=azure delete -f rook/rook-tools.yaml
kubectl --context=azure delete -f rook/rook-cluster.yaml
```

Check that the cluster namespace and all resources have been deleted

```
kubectl --context=azure get -n rook all
```

Cleanup rook cluster data

```
kubectl --context=azure apply -f rook/rook-cleanup-data-dangerous.yaml
```

Check that cleanup is complete

```
for p in $(kubectl --context=azure get pods -l app=rook-cleanup-data -o name); do echo -n "$p: "; kubectl --context=azure logs $p; done
```

Delete cleanup daemon set

```
kubectl --context=azure delete -f rook/rook-cleanup-data-dangerous.yaml
```

Delete Rook operator

```
kubectl --context=azure delete -f rook/rook-operator.yaml
```

Check that the operation namespace and all resources are deleted

```
kubectl --context=azure get -n rook-system all
```
