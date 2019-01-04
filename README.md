# Atomix Helm Charts

This project provides [Helm] charts for [Atomix].

* [Atomix](#atomix)
* [Atomix Operator](#atomix-operator)

## Atomix

The Atomix chart is implemented primarily as a `StatefulSet` with a headless
service for persistent pod identifiers. The chart supports:
* Any number of replicas
* Configurable pod disruption budget
* Optional host-based anti-affinity policy
* Configurable storage class
* Arbitrary Atomix configuration in HOCON or JSON format
* Upgrades through the `RollingUpdate` strategy

### Installation

The Atomix chart is not currently hosted anywhere. To install the chart,
clone the repository and `cd` into the `atomix-k8s` directory:

```
git clone --branch master https://github.com/atomix/atomix-helm.git && cd atomix-helm
```

To install the chart, run `helm install`:

```
helm install atomix
```

To install the chart on minikube, disable anti-affinity:

```
helm install --set podAntiAffinity.enabled=false atomix
```

### Configuration

#### Atomix cluster

The Atomix configuration is a set of root values used to override the
configuration for the Atomix pods themselves:
* `replicas` (default `3`)
* `logLevel` (default `INFO`)
* `heap` (default `2G`)
* `config`

The `config` field sets the base node configuration for all Atomix nodes.
The configuration format is HOCON and should exclude the `cluster` configuration,
which is managed by the chart itself. By default, the following configuration
is used:

```
managementGroup {
  type: raft
  name: system
  partitionSize: 3
  partitions: 1
  members: ${atomix.members}
}
partitionGroups.raft {
  type: raft
  name: raft
  partitionSize: 3
  partitions: ${atomix.replicas}
  members: ${atomix.members}
}
```

To override the configuration, create a new `yaml` file and provide it
to Helm when installing the chart, e.g.:

`atomix.yaml`

```yaml
config: |
  # Raft management group for strongly consistent primary elections
  managementGroup {
    type: raft
    partitionSize: 3
    partitions: 1
    members: ${atomix.members}
  }

  # Raft partition group for storing consistent primitives
  partititonGroups.consensus {
    type: raft
    partitionSize: 3
    partitions: ${atomix.replicas}
    members: ${atomix.members}
    storage.segmentSize: 32MB
    storage.maxEntrySize: 1KB
  }

  # Primary-backup data group for in-memory replication backed by Raft elections
  partitionGroups.data {
    type: primary-backup
    partitions: 31
  }

  # Define a default protocol
  defaultProtocol {
    type: multi-raft
    group: consensus
    readConsistency: SEQUENTIAL
    recoveryStrategy: RECOVER
  }

  # Sets the default protocol for DistributedSet, DistributedList, DistributedQueue
  primitiveDefaults.set.protocol: ${defaultProtocol}
  primitiveDefaults.list.protocol: ${defaultProtocol}
  primitiveDefaults.queue.protocol: ${defaultProtocol}

  # Simple primitive configuration to use a multi-primary protocol and
  # enable caching for all `AtomicMap`s named `foo`
  primitives.foo {
    type: atomic-map
    cache.enabled: true
    protocol {
      type: multi-primary
      group: data
      backups: 2
      maxRetries: 2
    }
  }
```

Some useful variables are provided by the chart for use in HOCON files:
* `${atomix.replicas}` - the configured number of replicas in the StatefulSet
* `${atomix.members}` - the list of member IDs in the StatefulSet for use in Raft partition group configurations

When used in the `atomix.config` configuration, Atomix will replace the provided
variables with the correct values for the running chart.

To install the chart with a custom configuration, use the `--values` option:

```
helm install --values=atomix.yaml atomix`
```

### image

The image used by the chart is configurable via the `image` object:
* `image.repository` (default `atomix/atomix`)
* `image.tag` (default `latest`)
* `image.pullPolicy` (default `IfNotPresent`)
* `image.pullSecrets` (default `[]`)

### podAntiAffinity

The `podAntiAffinity` object sets the options for the StatefulSet's
anti-affinity policy:
* `podAntiAffinity.enabled` (default `true`)

### podDisruptionBudget

The `podDisruptionBudget` configures the number of nodes that can be
taken offline by k8s:
* `podDisruptionBudget.enabled` (default `true`)
* `podDisruptionBudget.maxUnavailable` (default `1`)
* `podDisruptionBudget.minAvailable`

### persistence

The `persistence` object is used to override the persistence configuration
for the chart. The available `persistence` options are:
* `persistence.accessModes`
* `persistence.size` (default `10Gi`)
* `persistence.annotations` (default `{}`)
* `persistence.storageClass`

It's recommended that `local-storage` be used for clusters where Raft
partition groups are used. It's essential that Raft logs for a single pod
be stored on a single host.

## Upgrades

The Atomix chart supports upgrades through a simple `RollingUpdate` strategy.
To upgrade Atomix, use the `helm upgrade` command, overriding the `image.tag`
value:

```
helm install --set image.tag=3.0.6 atomix
helm upgrade --set image.tag=latest atomix
```

## Atomix Operator

The `atomix-operator` chart provides templates for deploying the
[Atomix Operator][Operator] to enable the deployment of custom `AtomixCluster`
resources. The operator is able to manage the Atomix cluster to provide for
much more involved cluster architectures, allowing partition groups to be added
to and removed from the cluster, facilitating benchmarking, and providing chaos
engineering features.

### Installation

The Atomix Operator chart is not currently hosted anywhere. To install the chart,
clone the repository and `cd` into the `atomix-helm` directory:

```
git clone --branch master https://github.com/atomix/atomix-helm.git && cd atomix-helm
```

To install the chart, run `helm install`:

```
helm install atomix-operator
```

Once the chart has been installed, you should be able to list `AtomixCluster`
resources:

```
kubectl get atomixclusters
```

### Configuration

The operator chart supports few configuration options since the majority of the
configurable options are provided through custom resources provided by the
operator itself.

### image

The image used by the chart is configurable via the `image` object:
* `image.repository` (default `atomix/atomix-operator`)
* `image.tag` (default `latest`)
* `image.pullPolicy` (default `IfNotPresent`)
* `image.pullSecrets` (default `[]`)

## Acknowledgements

Atomix is developed as part of the [ONOS][ONOS] project at the [Open Networking Foundation][ONF].

![ONF](https://3vf60mmveq1g8vzn48q2o71a-wpengine.netdna-ssl.com/wp-content/uploads/2017/06/onf-logo.jpg)

[Website]: https://atomix.io
[ONF]: https://opennetworking.org
[ONOS]: https://onosproject.org
[Atomix]: https://github.com/atomix/atomix
[Operator]: https://github.com/atomix/atomix-operator
[Helm]: https://helm.sh
