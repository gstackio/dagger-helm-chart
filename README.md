Dagger Helm chart
=================

This Helm chart is an evolution for running a Dagger build service on-premise.


Usage
-----

export the `_EXPERIMENTAL_DAGGER_RUNNER_HOST` environment variable with
[a value pointing to your target][conn_intrfc].

[conn_intrfc]: https://docs.dagger.io/manuals/administrator/custom-runner/#connection-interface

```shell
export _EXPERIMENTAL_DAGGER_RUNNER_HOST="kube-pod://$(kubectl get pod     --selector="name=dagger-dagger-helm-engine" --namespace="dagger"     --output=jsonpath='{.items[0].metadata.name}')?namespace=dagger"
```


Design
------

The default helm char as provided by Dagger ships with several issues.

- It uses a k8s `DaemonSet`, leading to Pods not being properly re-schduled on
  other nodes whenv their worker node evicts them.
- It uses `hostpath` volumes, which is bad practice.
- Cache volumes are distincts on each node, so that each Dagger Pod has a
  different build cache, leading to inconsistencies.
- The various Pods are not behind any k8s `Service`, leading to disruptions
  when Kubernetes nodes get updated, and no support for HA. When a node is
  stopped, its Dagger Pod is unavailable with no fallback solution.

This helm char adopts a different approach, yet very simple, leading to a more
sensible design:

- Based on a k8s `Deployment`.
- Behind a k8s `Service`.
- Using an ubiquitous NFS volume for cache, so that Pod share the same cache
  and may properly be stopped and restarted on any worker node.
