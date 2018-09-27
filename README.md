# Atomix Kubernetes

This project provides a [Helm] chart for [Atomix].

The chart is implemented primarily as a `StatefulSet` with a headless
service for persistent pod identifiers. The chart supports:
* Any number of replicas
* Configurable pod disruption budget
* Optional host-based anti-affinity policy
* Configurable storage class
* Arbitrary Atomix configuration in HOCON or JSON format
* Upgrades through the `RollingUpdate` strategy

## Installation

The Atomix chart is not currently hosted anywhere. To install the chart,
clone the repository and `cd` into the `atomix-k8s` directory:

```
git clone --branch master https://github.com/atomix/atomix-k8s.git && cd atomix-k8s
```

To install the chart, run `helm install`:

```
helm install .
```

## Configuration

### image

The image used by the chart is configurable via the `image` object:
* `repository` (default `atomix/atomix`)
* `tag` (default `latest`)
* `pullPolicy` (default `IfNotPresent`)
* `pullSecrets` (default `[]`)

### atomix

The `atomix` configuration is used to override the configuration for the
Atomix pods themselves:
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
atomix:
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
helm install --values=atomix.yaml .`
```

### podAntiAffinity

The `podAntiAffinity` object sets the options for the StatefulSet's
anti-affinity policy:
* `enabled` (default `true`)

### podDisruptionBudget

The `podDisruptionBudget` configures the number of nodes that can be
taken offline by k8s:
* `enabled` (default `true`)
* `maxUnavailable` (default `1`)
* `minAvailable`

### persistence

The `persistence` object is used to override the persistence configuration
for the chart. The available `persistence` options are:
* `accessModes`
* `size` (default `10Gi`)
* `annotations` (default `{}`)
* `storageClass`

It's recommended that `local-storage` be used for clusters where Raft
partition groups are used. It's essential that Raft logs for a single pod
be stored on a single host.

## Upgrades

The Atomix chart supports upgrades through a simple `RollingUpdate` strategy.
To upgrade Atomix, use the `helm upgrade` command, overriding the `image.tag`
value:

```
helm install --set image.tag=3.0.6 .
helm upgrade --set image.tag=latest .
```

## Acknowledgements

Atomix is developed as part of the [ONOS][ONOS] project at the [Open Networking Foundation][ONF].

![ONF](https://3vf60mmveq1g8vzn48q2o71a-wpengine.netdna-ssl.com/wp-content/uploads/2017/06/onf-logo.jpg)

[Website]: https://atomix.io
[ONF]: https://opennetworking.org
[ONOS]: https://onosproject.org
[Atomix]: https://github.com/atomix/atomix
[Helm]: https://helm.sh
