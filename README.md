# Upgrade MongoDB clusters using Ansible

This tool currently supports deployments on Red Hat-based distributions, including Amazon Linux 2023. The MongoDB version is determined by the [Percona Release](https://docs.percona.com/percona-software-repositories/percona-release.html), and the availability of MongoDB versions depends on the operating system version. The upgrade installs the latest available version from the given release, starting from psmdb-60. This means automated upgrades are available for clusters running MongoDB 5.0 or later. If your cluster is running a version prior to 5.0, you must manually upgrade it before utilizing the automated upgrade process.

The upgrade process checks all components of the cluster and will produce warnings and abort the upgrade if any component is running a version older than 1 version prior to the version you are trying to upgrade.  In other words, you can not skip major versions during an upgrade (you can't upgrade from 5.0 to 7.0, you would have to go from 5.0 to 6.0 and so on.)

The upgrade process verifies the cluster's health and each node's state and it will only proceed with the upgrade if all nodes are healthy and there are no issues with any host. If the upgrade detects any node state other than `PRIMARY`, `SECONDARY` or `ARBITER` it will not proceed with the upgrade.

To upgrade a MongoDB cluster, each replicaset must have at least 3 healthy nodes (non cfg nodes), which can be in any of the states: PRIMARY, SECONDARY, or ARBITER. The automated upgrade process will verify that each replicaset meets this requirement in order for the cluster to remain operational during the upgrade. This is to prevent replica sets with only two nodes from being shut down during the upgrade, as doing so would prevent the cluster from electing a new primary, potentially causing downtime. There is no minimum number of cfg nodes required, as long as they are all healthy.

Outages are not expected during the upgrade process; however, depending on system load, some minor disruptions (e.g., latency spikes) may be observed. Although the MongoDB driver includes a retry mechanism, prolonged stepDown events can still manifest as latency from the application's perspective. Each node is upgraded sequentially in a controlled manner: secondary nodes are upgraded first, followed by the primary node in each replica set. Keep in mind certain applications may be more sensitive to this behavior.

#### As always, we strongly recommend testing the upgrade in a non-production environment first to ensure compatibility with your specific application workload.

Since this is an automated process, it assumes that you have already completed the necessary due diligence to ensure a smooth and successful upgrade. Although this is an automated process, you are still responsible for choosing the best time of day to perform the upgrade based on your workload, in addition to following the recommendations below:

1. Review MongoDB Version Compatibility
2. Check for Deprecated Features or Parameters
3. Check System Resources
4. Ensure Compatibility with MongoDB Drivers
5. Plan for Downtime or Maintenance Window
6. Check for Indexes or Schema Changes (if applicable)
7. Ensure Sufficient Disk Space for Upgrade
8. Review Authentication and Security Settings Changes (if applicable)

As mentioned above, in order to minimize any downtime, the automated process upgrades the cluster in a "rolling" fashion:

1. Health checks and FCV configured (this is done for both replicaset and sharded clusters)

For replicaset (NON sharded cluster)

1. Upgrade 1 `SECONDARY` node at a time
2. Once all `SECONDARY` nodes are upgraded, the `PRIMARY` is stepped down and upgraded
3. Upgrade is complete

For a sharded cluster

1. Disable balancers
2. Upgrade 1 `SECONDARY` cfg node at a time 
3. Once all `SECONDARY` cfg nodes are upgraded, the `PRIMARY` cfg node is stepped down and upgraded
4. Upgrade all shards. Each shard is upgraded in a rolling fashion. The automated process selects 1 `SECONDARY` node from each existing shard and it upgrades these nodes simulataneously (i.e. If you have 3 shards with 5 nodes each the process will upgrade 3 nodes at the same time, 1 node from each shard --- shard1node5, shard2node5, shard3node5 ... etc ...)
5. Once all `SECONDARY` nodes for the given shard are upgraded, stepdown the `PRIMARY` and upgrade it
6. Upgrade 1 Mongos router at a time to prevent outages
7. Re-enable balancers
8. Upgrade is complete

The upgrade does not set the feature compatibility version (FCV) to the new release. Enabling these backwards-incompatible features can complicate the downgrade process since you must remove any persisted backwards-incompatible features before you downgrade. It is recommended that after upgrading, you allow your deployment to run without enabling these features for a burn-in period to ensure the likelihood of downgrade is minimal. When you are confident that the likelihood of downgrade is minimal, enable these features.

## Quick guide
1. Make the necessary changes according to your environmen to `inventory`  
2. Rename the template file `group_vars/all_template` to `group_vars/all` and edit the file with the appropriate options, passwords, ports, etc.
3. Make sure ansible can connect to your cluster

```
cd ./ansible
ansible -i inventory all -m ping
```

5. Run the `main.yml` playbook pointing Ansible to the desired inventory file. For example:

```
ansible-playbook -i inventory main.yml
```

The upgrade process takes around 15 min for a cluster with the following specs:

3 cfg nodes
2 shards of 3 nodes each
1 mongos


## Inventory file

- Keep a separate inventory file per environment (for example dev, test, prod).
- If you have more than one cluster per environment, then keep one inventory per cluster as well (for example devrs1, devrs2, devshard1, devshard2).
- On each inventory file we have to specify groups for each shard's replicaset. You have to name them as such, do not change the naming syntax prefix of the groups. 
The group names do not affect your replicaset names, these are just used by ansible.
  - `shardXXXX` for the shard nodes
  - `cfg` for the config nodes
  - `mongos` for the mongos routers

- For each standalone replicaset you have to name them `rsXXXX` (don't use `shardXXXX` for replica sets not part of a sharded cluster).
- Arbiters are specified by adding `arbiter=True` tag to each arbiter host

- Example of an inventory for a sharded cluster:
```
[cfg]
dev-mongo-cfg00 
dev-mongo-cfg01
dev-mongo-cfg02

[shard0]
dev-mongo-shard00svr0 
dev-mongo-shard00svr1
dev-mongo-shard00svr2 arbiter=True

[shard1]
dev-mongo-shard01svr0 
dev-mongo-shard01svr1
dev-mongo-shard01svr2 arbiter=True

[mongos]
dev-mongo-router00

[all:vars]
ansible_ssh_user=your_ssh_user_here
```

- Example of an inventory for a single replicaset:
```
[rs1]
host11 
host12
host13

[all:vars]
ansible_ssh_user=your_ssh_user_here
```

## Configuration
* The all variables file (`group_vars/all`) contains all the user-modifiable parameters. Each of these come with a small description to clarify the purpose, unless it is self-explanatory. 
You have to review and modify this file before running the upgrade, as the configuration is specific to each environment.

## FCV (Feature Compatibility Version)

The cluster must have the `featureCompatibilityVersion` set to the current MongoDB major version before an upgrade can be performed. The automated process handles this step for you.
For example, if you are running MongoDB 6.0.x and upgrading to 7.0, the automated process will set the `featureCompatibilityVersion` to "6.0" before proceeding with the upgrade.

## Running
#### The playbook is meant to handle upgrades without requiring any special flags:

* Upgrade cluster 
```
ansible-playbook -i inventory main.yml
```

#### Some tasks can be done individually, although NOT recommended.

* Set Feature Compatibility Version
```
ansible-playbook -i inventory main.yml --tags "upgrade:fcv"
```

* Stop balancers 
```
ansible-playbook -i inventory main.yml --tags "upgrade:lbStop"
```

* Start balancers
```
ansible-playbook -i inventory main.yml --tags "upgrade:lbStart"
```

* If for some reason you only want to upgrade a given host or group of hosts, you can provide the group name or host name:
```
ansible-playbook -i inventory main.yml --limit mongos
```

```
ansible-playbook -i inventory main.yml --limit dev-mongo-router00
```


## TLS Setup

Remember to set `use_tls: true` in the variables file as well as any certs required (this is set to `false` by default)

