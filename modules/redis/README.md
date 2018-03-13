**Note**: This public repo contains the documentation for the private GitHub repo <https://github.com/gruntwork-io/module-cache>.
We publish the documentation publicly so it turns up in online searches, but to see the source code, you must be a Gruntwork customer.
If you're already a Gruntwork customer, the original source for this file is at: <https://github.com/gruntwork-io/module-cache/blob/master/modules/redis/README.md>.
If you're not a customer, contact us at <info@gruntwork.io> or <http://www.gruntwork.io> for info on how to get access!

# Redis Module

This module creates an ElastiCache cluster that runs [Redis](http://redis.io/).

The redis cluster is managed by AWS and automatically detects and replaces failed nodes, streamlines software upgrades
and patches, enables easy scaling of the cluster both horizontally (add more nodes) and vertically (increase the power
of existing nodes), and simplifies backup and restore functionality.

## About Amazon ElastiCache

### What is Amazon ElastiCache?

Before [Amazon ElastiCache](http://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/WhatIs.html) existed, teams would painstakingly configure the memcached or redis caching engines on their own. Setting up automatic failover, read replicas, backups, and handling upgrades are all non-trivial and AWS recognized they could implement these features according to best practices themselves, sparing customers the time and cost of doing it themselves.

Behind the scenes, ElastiCache runs on EC2 Instances located in subnets and protected by security groups you specify.

### Structure of an ElastiCache Redis deployment

- **Nodes:** The smallest unit of an ElastiCache Redis deployment is a node. It's basically the network-attached RAM on which the cache engine (Redis in this case) runs

- **Shards:** Also sometimes called "Node Group". A Shard is a replication-enabled collection of multiple nodes. Within a Shard, one node is the primary read/write node while the rest are read-only replicas of the primary node. A Shard can have up to 5 read-only replicas.

- **Cluster:** An ElastiCache cluster is a collection of one or more Shards. Somewhat confusingly, an ElastiCache Cluster has a "cluster mode" property that allows a Cluster to distribute its data over multiple Shards. When cluster mode is disabled, the Cluster can have at most one Shard. When cluster mode is enabled, the Cluster can have up to 15 Shards. A "cluster mode disabled" Cluster (i.e. Single Shard cluster) can be scaled horizontally by adding/removing replica nodes within its single Shard, vertical scaling is achieved by simply changing the node types. However, a "cluster mode enabled" Cluster (i.e. Multi Shard cluster) can be scaled horizontally by adding/removing Shards, the node types in the Shards can also be changed to achieve vertical scaling. Each cluster mode will be explained in detail below with additional info on when each one will be more appropriate depending on your scaling needs.

### How do you connect to the Redis Cluster?

- When connecting to Redis (cluster mode disabled) from your app, direct all operations to the **Primary Endpoint** of the **Cluster.** This way, in the event of a failover, your app will be resolving a DNS record that automatically gets updated to the latest primary node.

- When connecting to Redis (cluster mode enabled) from your app, direct all reads/writes to the **Configuration Endpoint** of the **Cluster.** Ensure you have a client that supports Redis Cluster (redis 3.2). You can still read from individual node enpoints.

- In both "cluster mode enabled" and "cluster mode disabled" deployment models you can still direct reads to any of the **Read Endpoints** of the nodes in the Cluster, however you now risk reading a slightly out-of-date copy of the data in the event that you read from a node before the primary's latest data has synced to it.

This module outputs [Terraform output variables](https://www.terraform.io/intro/getting-started/outputs.html) that contain the address of the primary endpoint and read endpoints. You can programmatically extract these variables in your Terraform templates and pass them to other resources (e.g. as environment variables in an EC2 Instance) You'll also see the variables at the end of each `terraform apply` call or if you run `terraform output`.

### How do you scale the Redis Cluster?

You can scale your ElastiCache Cluster either horizontally (by adding more nodes) or vertically (by using more powerful nodes), but the method depends on whether Cluster Mode is enabled or disabled.

#### When Cluster Mode is Disabled

This mode is useful when you'd prefer to have only a single point for data to be written to the redis database. All data is written to the primary node of the single shard which is now replicated to the replica nodes. The advantage of this approach is that you're sure that all your data is present at a single point which could make migrations and backups a lot easier, if there's a problem with the primary write node however, all write attempts will fail.

- **Vertical:** You can increase the type of the read replica nodes using the `instance_type` parameter (see [here(https://aws.amazon.com/elasticache/details/#Available_Cache_Node_Types) for valid values).

- **Horizontal:** You can add up to 5 replica nodes to a Redis Cluster using the `cluster_size` parameter. There is always a primary node where all writes take place, but you can reduce load on this primary node by offloading reads to the non-primary nodes, which are known as Read Replicas.

For more info on both methods, see [Scaling Single-Node Redis (cluster mode disabled) Clusters](https://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/Scaling.RedisStandalone.html).

#### When Cluster Mode is Enabled

When cluster mode is enabled, data you write is split among the primary nodes in multiple Shards, and each data stored in those Shards are replicated among the read replicas. If one Shard becomes unavailable for whatever reason, other shards are still available for writing.

To deploy a Redis "cluster mode enabled" cluster you must ensure that the `enable_automatic_failover` parameter is set to `true` and the `cluster_mode` variable has a single map with `num_shards` and `replicas_per_shard` parameters.

E.g. the following Terraform code specifies a "cluster mode enabled" Cluster with 2 Shards and 1 Read Replica per Shard

```hcl
cluster_mode = [{
    num_node_groups = 2
    replicas_per_node_group = 1
}]
```

- **Horizontal:** A "cluster mode enabled" cluster can be scaled horizontally by adding more Shards - also called Resharding, the amount of read replicas present in the extra Shards is the same as the number specified when the cluster was originally created. Resharding can take anywhere from a few minutes to several hours.

- **Vertical:** A "cluster mode enabled" cluster can be scaled vertically by changing the node types within each Shard. This method is offline only and the cluster has to be backed up, a new cluster created with the required node types and the backup restored to that new cluster. During this process your entire cluster will experience a downtime which could last for any length of time depending on how much data the original cluster contained.

For more info on scaling "cluster mode enabled" Redis clusters, see [Scaling Multishard Redis (cluster mode disabled) Clusters](https://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/scaling-redis-cluster-mode-enabled.html)

## How do you use this module?

See the [redis example](/examples/redis) for an example.

### ElastiCache Old Terminology

Prior to October 2016, ElastiCache for Redis did not support [partitioning](http://redis.io/topics/partitioning)
(aka sharding) in Redis and as a result used a bunch of confusing terminologies to explain setting up a Redis cluster.

- **Cluster:** In ElastiCache, each Redis node is labeled its own "cluster", despite that there is only a single node in the cluster. This is an artifact of how ElastiCache implements memcached clusters, which can support multiple nodes.

- **Replication Group:** The Redis "read/write primary cluster" (the master node) will asynchronously copy its data to all other Redis clusters (individual nodes) in the same Replication Group. In a sense, the "real" redis cluster in ElastiCache is actually the Replication Group.

- **Primary Cluster:** In a Replication Group, the primary cluster is where all write traffic is directed. Initially, you can direct all your reads here as well.

- **Read Replicas:** In a Replication Group, data written to the primary cluster is asynchronously copied to each Read Replica. These are useful both to reduce the read load on the primary cluster and to faciliate fast failover.

- **Multi-AZ:** Using the Multi-AZ option means that if the primary cluster fails, ElastiCache will auto failover to a Read Replica. Without this option, the primary cluster node will simply replace itself, which will take longer and may result in additional data loss.

### Common Gotcha's

- In the event of a Redis failover, *you will experience a small period of time during which writes are not accepted*, so make sure your app can handle this gracefully. In fact, consider simulating Redis failovers on a regular basis or with automated testing to validate that your app can handle it correctly.

- Test that your app does not cache the DNS value of the Primary Endpoint! Java, in particular, has undesirable defaults around DNS caching in many cases. If your code does not honor the TTL property of the Primary Endpoint's DNS record, then your app may fail to reach the new primary node in the event of a failure.

- The only way to add more storage space to nodes in a Redis Cluster (cluster mode disabled) is to scale up to a larger node type.

