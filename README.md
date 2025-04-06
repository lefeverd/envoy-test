# Envoy

This repository contains some test files to test envoy dynamic configuration setup, using a configmap.

## Create a test cluster (kind)

`kind create cluster -n envoy-test --config kind-cluster.yaml`

## Deploy ingress-nginx

`kubectl apply -f deploy-ingress-nginx.yaml`

Only the deployment spec was updated to add a nodeSelector :

```
nodeSelector:
    kubernetes.io/hostname: "envoy-test-control-plane"
```


## Filesystem subscriptions

As stated in the [documentation](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#filesystem-subscriptions),
The simplest approach to delivering dynamic configuration is to place it at a well known path specified in the ConfigSource.

For this, we create a very simple [configmap](./envoy/envoy-cm-dynamic.yaml) which defines the paths (in the `/config/` directory) to the dynamic resources for Listeners (LDS) and Clusters (CDS).  
This configmap is mounted at `/etc/envoy/envoy.yaml` and used as configuration file when starting envoy.

We can then define the LDS and CDS configuration in another [configmap](./envoy/envoy-cm-lds-cds.yaml), and mount it in the `/config` directory inside the container.

## Hot reload from configmap

Not working yet, see https://ahmet.im/blog/kubernetes-inotify/.
The files are updated in the pod (symlink is updated to point to the new version) but envoy does not reload.

## Notes

When using dynamic configuration, it looks like the node id and cluster are mandatory :

```
node:
    id: envoy-1
    cluster: envoy-test
```

You must also, in the LDS and CDS configuration, declare the `@type` of each resource, for instance `"@type": type.googleapis.com/envoy.config.listener.v3.Listener`.
