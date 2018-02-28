### 1.3. Setting up demo environment with Kublr command line utilities

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
