Dagger Helm chart
=================

This Helm chart is an experiment for running a Dagger build service on-premise
in Kubernetes, so that is can be leveraged from a Concourse CI system. See the
[learnings and further work](#learnings-and-further-work) section for reasons
why the result is not yet ready for being used.


Rationale
---------

The design of the [default Helm chart][dagger_helm_chart] is to run a
`DaemonSet` of Pods with `hostPath` local volumes, and use a `kube-pod://` URL
sheme for the `_EXPERIMENTAL_DAGGER_RUNNER_HOST` environment variable so that
the `dagger` CLI can access the build service.

Here the idea is to evolve towards a k8s `Deployment`, exposed by a k8s
`Service`, and use a `tcp://` URL scheme for the
`_EXPERIMENTAL_DAGGER_RUNNER_HOST` environment variable. For the `hostPath`
volume mounted to `/var/lib/dagger`, the idea is to replace it with a shared
`nfs` volume.

[dagger_helm_chart]: https://github.com/dagger/dagger/tree/main/helm/dagger


### Details

#### Default Dagger Helm chart design

The default Helm chart as provided by Dagger ships with a design that is
subject to discussion.

- It uses a k8s `DaemonSet`, leading to Pods not being properly re-schduled on
  other nodes whenv their worker node evicts them.
- It uses `hostpath` volumes, which is not recommended practice.
    - For `/var/lib/dagger`, to store some ephemeral storage, basically
      storing the k8s node's cache for Dagger builds.
    - For `/var/run/dagger` to expose the `buildkitd.sock` socket. For which
      goal? This is unclear.
- Cache volumes are distincts on each node, so that each Dagger Pod has a
  different build cache. This might be improved with a shared cache, for
  consistent build performance and results on all nodes.
- The various Pods are not behind any k8s `Service`. This is basically fine
  when Dagger Pods are accessed by GitLab or GitHub Action runners, co-located
  on each k8s node. But when in need for accessing those Dagger Pods by some
  expernal CI system like Concourse, this no more relevant.
- A `ServiceAccount` may be created when the `engine.newServiceAccount.create`
  value is set to `true`, and an `existingServiceAccount` might even be
  declared, but the reason for this service account is unclear. It looks like
  dead code.
- The way the dagger service is to be reached out to is quite complicated:

    ```shell
    export _EXPERIMENTAL_DAGGER_RUNNER_HOST="kube-pod://$(kubectl get pod \
        --selector="name=dagger-dagger-helm-engine" --namespace="dagger" \
        --output=jsonpath='{.items[0].metadata.name}')?namespace=dagger"
    ```


#### Improved Dagger Helm chart design

This helm chart adopts a different approach, yet very simple, leading to a
more sensible design:

- Based on a k8s `Deployment`.
- Behind a k8s `Service`.
- Using an ubiquitous NFS volume for cache, so that Pod share the same cache
  and may properly be stopped and restarted on any worker node.


Learnings and further work
--------------------------

### Learnings

The performance of the `/var/lib/dagger` volume needs to be pretty high,
because ephemeral containers filesystems are created there. When using an
`nfs` volume here, the performances are severely degraded.

There is a `/var/lib/dagger/buildkitd.lock` lock that prevents the
`/var/lib/dagger` volume to be shared between many Dagger Pods.


### Further work

- Understand the reasons behind the `/var/lib/dagger/buildkitd.lock` and get
  rid of it.
- Adopt performance-friendly local volumes, like a `zfs` storage with OpenEBS.
- Configure the `[grpc.tls]` section of the `buildkitd.toml` configuration, as
  allowed by the `engine.config` value, so that TCP traffic is properly
  encrypted and Dagger clients properly authenticated.


Usage
-----

Before you start, ensure you have a properly setup `kubectl` cluter, user, and
context. As a result, `kubectl version --short` should run fine.

Clone the Helm chart repository and install it.

```shell
git clone https://github.com/gstackio/dagger-helm-chart.git
helm upgrade --install --namespace=dagger --create-namespace dagger dagger-helm-chart/helm/dagger/
```

Export the `_EXPERIMENTAL_DAGGER_RUNNER_HOST` environment variable with
[a value pointing to your target][conn_intrfc].

```shell
export _EXPERIMENTAL_DAGGER_RUNNER_HOST="tcp://dagger-dagger-helm-service.dagger.svc.cluster.local:2555"
```

[conn_intrfc]: https://docs.dagger.io/manuals/administrator/custom-runner/#connection-interface


Authors and license
-------------------

Copyright © 2024-present, Benjamin Gandon, Gstack

This Dagger Helm Chart is released under the terms of the
[Apache 2.0 license](http://www.apache.org/licenses/LICENSE-2.0).
