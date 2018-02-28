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

### 1.2. Non-Kublr created clusters

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

    See section **1.4. Variations** for more details.

### 1.3. Setting up demo environment with Kublr

#### 1.3.1. Configure access credentials

If Kublr is used to setup the cluster, it is expected that the user has
access to Kublr private docker and Helm repositories.

To make sure that set access up

1.  Ensure that you are logged in to Kublr docker repository at
    `docker.ecp.eastbanctech.com`

    ```
    docker login docker.ecp.eastbanctech.com
    ```

2.  Ensure that environment variables to access Kublr helm repository at
    `nexus.ecp.eastbanctech.com` are setup correctly.

    ```
    export HELM_REPO_USERNAME=...
    export HELM_REPO_PASSWORD=...
    ```

3.  An environment variable with Kublr version is set.

    ```
    export KUBLR_VERSION=...
    ```

4.  Copy `cluster-aws/in/input-portability-aws-template.yaml` into
    `cluster-aws/in/input-portability-aws.yaml` and
    `cluster-azure/in/input-portability-azure-template.yaml` into
    `cluster-azure/in/input-portability-azure.yaml`.

    In the copied files `cluster-aws/in/input-portability-aws.yaml` and
    `cluster-azure/in/input-portability-azure.yaml` uncomment the
    commented lines and provide correct values according to Kublr
    documentation and desired configuration.

#### 1.3.1. Generate cluster artifacts

1.  For AWS cluster:

    ```
    mkdir -p "$(pwd)/cluster-aws/out"
    docker run \
        -u "$(id -u):$(id -g)" \
        -v "$(pwd)/cluster-aws/in:/gen" \
        -v "$(pwd)/cluster-aws/out:/gen-out" \
        -v "${HOME}/.aws:/.aws" \
        -e HOME=/ \
        docker.ecp.eastbanctech.com/kublr/gen:${KUBLR_VERSION} \
            -f /gen/input-portability-aws.yaml \
            -o /gen-out
    ```

2.  For Azure cluster:

    ```
    mkdir -p "$(pwd)/cluster-azure/out"
    docker run \
        -u "$(id -u):$(id -g)" \
        -v "$(pwd)/cluster-azure/in:/gen" \
        -v "$(pwd)/cluster-azure/out:/gen-out" \
        -e HOME=/ \
        docker.ecp.eastbanctech.com/kublr/gen:${KUBLR_VERSION} \
            -f /gen/input-portability-azure.yaml \
            -o /gen-out
    ```

#### 1.3.2. Create clusters

1. AWS:

    ```
    ( cd cluster-aws/out; bash aws-portability-aws-aws1.sh )
    ```

    Cluster config will be automatically downloaded by the script:
    `cluster-aws/out/config-portability-aws.yaml`

2.  Azure:

    ```
    ( cd cluster-azure/out; bash azure-portability-azure-azure1-deploy.sh )
    ```

    Download cluster config from Azure deployment:
    `cluster-azure/out/config-portability-azure.yaml`

#### 1.3.3. Deploy features

1.  AWS

    -   kublr-system

        ```
        KUBECONFIG=cluster-aws/out/config-portability-aws.yaml \
        helm upgrade -i \
            --namespace kube-system \
            kublr-system \
            https://${HELM_REPO_USERNAME}:${HELM_REPO_PASSWORD}@nexus.ecp.eastbanctech.com/repository/helm/kublr-system-0.2.3.tgz \
            -f cluster-aws/out/kublr-system-values.yaml
        ```

    -   kublr-feature-ingress

    ```
    KUBECONFIG=cluster-aws/out/config-portability-aws.yaml \
    helm upgrade -i \
        --namespace kube-system \
        kublr-feature-ingress \
        https://${HELM_REPO_USERNAME}:${HELM_REPO_PASSWORD}@nexus.ecp.eastbanctech.com/repository/helm/kublr-feature-ingress-0.3.1.tgz \
        -f clusters/kublr-feature-ingress-values.yaml
    ```

    Point `*.port-aws.demo.kublr.com` at the ELB created for the ingress
    controller.

2.  Azure

    -   kublr-system

        ```
        KUBECONFIG=cluster-azure/out/config-portability-azure.yaml \
        helm upgrade -i \
            --namespace kube-system \
            kublr-system \
            https://${HELM_REPO_USERNAME}:${HELM_REPO_PASSWORD}@nexus.ecp.eastbanctech.com/repository/helm/kublr-system-0.2.3.tgz \
            -f clusters/kublr-system-values.yaml
        ```

    -   kublr-feature-ingress

        ```
        KUBECONFIG=cluster-azure/out/config-portability-azure.yaml \
        helm upgrade -i \
            --namespace kube-system \
            kublr-feature-ingress \
            https://${HELM_REPO_USERNAME}:${HELM_REPO_PASSWORD}@nexus.ecp.eastbanctech.com/repository/helm/kublr-feature-ingress-0.3.1.tgz \
            -f kublr-feature-ingress-values.yaml
        ```

        Point `*.port-azure.demo.kublr.com` at the IP of the load balancer
        created for the ingress controller.

#### 1.3.4. Other

Merge files `cluster-aws/out/config-portability-aws.yaml` and
`cluster-azure/out/config-portability-azure.yaml` into a single
kubernetes config file `clusters/config-portability.yaml`

Make sure that contexts in the file are renamed to `aws` and `azure` for
AWS and Azure clusters correspondingly, and current context is not
defined.

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

Test AWS load balancer: `http://<aws-elb-specific-address>.elb.amazonaws.com`
(this address will be unique for your specific cluster).

Test Azure load balancer: `http://<azure-lb-specific-ip>` (this address will be
unique for your specific cluster).

Both should open default 404 page.

### 1.4. Variations

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
Ceph cluster and `replicapool` replica pool in that cluster

#### 2.3.1. Deploy Rook operator for Ceph cluster

Make sure node labels are correct

```
for n in $(kubectl --context=azure get nodes -o=custom-columns=NAME:.metadata.name --no-headers); do echo -n "$n: "; kubectl --context=azure label --overwrite node "$n" "kubernetes.io/hostname=$n"; done
```

Deploy operator

```
kubectl --context=azure apply -f rook/rook-operator.yaml

# Check that it is deployed (1 operator, 1 agent per node should be available)
kubectl --context=azure get -n rook-system pods
```

Deploy cluster and tools

```
# cluster
kubectl --context=azure apply -f rook/rook-cluster.yaml

# check deployed (1 api, 1 mgr, 3 mon, 1 osd per node)
kubectl --context=azure get -n rook pods

# tools
kubectl --context=azure apply -f rook/rook-tools.yaml

# test cluster and tools
kubectl --context=azure exec -n rook rook-tools -- rookctl status
```

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
