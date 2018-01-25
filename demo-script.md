# Demo script

Show helm package

Explain the demo application structure:

-   WP Ingress

-   WP Service

-   WP Deployment

-   MS Service

-   MS Deployment

-   MS PVC

----------------------------------------------------------------------------------------------------------------------------------------------------------------

Show clusters

-   AWS:

    `https://<aws-cluster-api-endpoint>/ui`

-   Azure:

    `https://<azure-cluster-api-endpoint>/ui`

----------------------------------------------------------------------------------------------------------------------------------------------------------------

Deploy evaluation on AWS

Show values-evaluation.yaml

```
helm --kube-context=aws upgrade -i demo demo-wordpress -f values-evaluation.yaml
```

1-2m

Start port forwarding:

```
kubectl --context aws port-forward \
    $(kubectl --context aws get pods -l app=demo-demo-wordpress-wordpress -o custom-columns=name:metadata.name --no-headers=true) \
    8080:80
```

Open localhost:8080

1m

----------------------------------------------------------------------------------------------------------------------------------------------------------------

Deploy with Ingress and SSL termination

```
helm --kube-context=aws upgrade -i demo demo-wordpress -f values-evaluation-ingress-ssl-host-aws.yaml
```

Show values-evaluation-ingress-ssl-host-aws.yaml

Using Ingress with SSL termination and automatic certificate acquisition through Letsencrypt using LEGO

Still no persistence, ephemeral storage

Demo Wordpress access via http://wp.port-aws.demo.kublr.com/

Gets redirected to https://wp.port-aws.demo.kublr.com/ with correct certificate

2m

----------------------------------------------------------------------------------------------------------------------------------------------------------------

On certificate management

```
helm --kube-context=aws upgrade -i demo demo-wordpress -f values-evaluation-ingress-ssl-host-aws.yaml --set ingress.host=demo1.port-aws.demo.kublr.com
```

Open http://demo1.port-aws.demo.kublr.com/

Gets redirected to https://demo1.port-aws.demo.kublr.com/ with correct certificate

2m

----------------------------------------------------------------------------------------------------------------------------------------------------------------

Deploy AWS with EBS persistence:

```
helm --kube-context=aws upgrade -i demo demo-wordpress -f values-persistent-ingress-ssl-host-aws.yaml
```

Show values-persistent-ingress-ssl-host-aws.yaml

Using Ingress with SSL termination and automatic certificate acquisition through Letsencrypt using LEGO

Persistence is based on dynamically allocated EBS

In UI see PV, PVC, and restarted MySql pods

Open http://wp.port-aws.demo.kublr.com/

Error because DB has been recreated

Delete wordpress pod

Open http://wp.port-aws.demo.kublr.com/

Init site

Login

show works

Kill mysql

show that it continues working

3m

----------------------------------------------------------------------------------------------------------------------------------------------------------------

Show slide with RDS

Show values-persistent-rds-ingress-ssl-host-aws.yaml

Explain using external RDS (no demo)

1m

----------------------------------------------------------------------------------------------------------------------------------------------------------------

Using Ingress with SSL termination and automatic certificate acquisition through Letsencrypt using LEGO

Persistence is based on dynamically allocated Ceph disk image in a self-hosted Ceph cluster

Deploy Rook operator to Azure

```
kubectl --context=azure apply -f rook/rook-operator.yaml
```

Check that it is deployed (1 operator, 1 agent per node should be available)

```
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

Deploy demo application with Ceph persistence to AWS

```
helm --kube-context=azure upgrade -i demo demo-wordpress -f values-persistent-ingress-ssl-host-azure.yaml
```

5m - switch to the next slide for some time (3-5m)

Check that the application is working

Open http://wp.port-azure.demo.kublr.com/

Review objects in K8S UI - app, Ceph cluster, and Rook Ceph operator
