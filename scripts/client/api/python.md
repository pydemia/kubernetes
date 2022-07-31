# Kubernetes API

https://github.com/kubernetes-client/python

## Installation

```bash
pip install kubernetes
```

## Client with `kubeconfig`

```py
from kubernetes import client, config, watch

# Configs can be set in Configuration class directly or using helper utility
config.load_kube_config()

k8s_client = client.CoreV1Api()
```

## Basic Usage

* without `watch`

```bash
kubectl get pods --all-namespace
```

is equivalent to:

```py
res = k8s_client.list_pod_for_all_namespaces(watch=False)
for i in res.items:
    print("%s\t%s\t%s" % (i.status.pod_ip, i.metadata.namespace, i.metadata.name))
```

* with `watch`

```bash
kubectl get pods --all-namespace
```

```py
w = watch.Watch()
for event in w.stream(k8s_client.list_namespace, _request_timeout=60):
    print("Event: %s %s" % (event['type'], event['object'].metadata.name))
    count -= 1
    if not count:
        w.stop()

```
