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
The way kubelet updates a ConfigMap is not by just updating it, but by creating a new versioned directory, and atomically update the symlink (see https://ahmet.im/blog/kubernetes-inotify/).  
For that reason, we need to watch the complete `/config` directory, using `watch_directory`, as [explained in the docs](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/config_source.proto#config-core-v3-pathconfigsource).

## Notes

When using dynamic configuration, it looks like the node id and cluster are mandatory :

```
node:
    id: envoy-1
    cluster: envoy-test
```

You must also, in the LDS and CDS configuration, declare the `@type` of each resource, for instance `"@type": type.googleapis.com/envoy.config.listener.v3.Listener`.
