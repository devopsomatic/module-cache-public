**Note**: This public repo contains the documentation for the private GitHub repo <https://github.com/gruntwork-io/module-cache>.
We publish the documentation publicly so it turns up in online searches, but to see the source code, you must be a Gruntwork customer.
If you're already a Gruntwork customer, the original source for this file is at: <https://github.com/gruntwork-io/module-cache/blob/master/modules/redis/README.md>.
If you're not a customer, contact us at <info@gruntwork.io> or <http://www.gruntwork.io> for info on how to get access!

# Redis Module

This module creates an ElastiCache cluster that runs [Redis](http://redis.io/).

The redis cluster is managed by AWS and automatically detects and replaces failed nodes, streamlines software upgrades
and patches, enables easy scaling of the cluster both horizontally (add more nodes) and vertically (increase the power
of existing nodes), and simplifies backup and restore functionality.

#### Terraform Limitations with ElastiCache

*NOTE: Currently, Terraform only supports replication and multi-AZ clusters for memcached and not Redis, so we must
make use of the [aws_cloudformation_stack](https://www.terraform.io/docs/providers/aws/r/cloudformation_stack.html)
terraform resource type to create the Redis replication group.*

*Once Terraform resolves a few bugs (see [#4361](https://github.com/hashicorp/terraform/issues/4361),
[#4363](https://github.com/hashicorp/terraform/pull/4363), [#4946](https://github.com/hashicorp/terraform/issues/4946),
and [#6836](https://github.com/hashicorp/terraform/pull/6836)), we can remove the `aws_cloudformation_stack` resource
and use native terraform resource types only.*

## About Amazon ElastiCache

### What is Amazon ElastiCache?

Before [Amazon ElastiCache](http://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/WhatIs.html) existed, teams
would painstakingly configure the memcached or redis caching engines on their own. Setting up automatic failover, read
replicas, backups, and handling upgrades are all non-trivial and AWS recognized they could implement these features
according to best practices themselves, sparing customers the time and cost of doing it themselves.

Behind the scenes, ElastiCache runs on EC2 Instances located in subnets and protected by security groups you specify.

### How does clustering work in ElastiCache for Redis?

An important limitation of ElastiCache for Redis is that there is no support for [partitioning](http://redis.io/topics/partitioning)
(aka sharding) in Redis. Instead ElastiCache for Redis supports only a single "primary node" where all writes (and
optionally reads) are directed. Confusingly, ElastiCache calls each individual redis node a "Cache Cluster", despite
that this cluster can only ever have exactly one node.

ElastiCache does support "Read Replicas", which are nodes that asynchronously receive a copy of the data written to the
primary node. The Read Replicas and primary node are all grouped together into a "Replication Group".

If the primary node were to fail abd the `enable_multi_az` Terraform variable is set to `true`, ElastiCache will
automatically promote a Read Replica to be the new primary node. Note that any data that was written to the primary
node and not yet copied to the read replica will be lost, however this is likely to be a very small amount of data.

### How do you connect to the Redis Replication Group?

- When connecting to Redis from your app, direct all reads/writes to the **Primary Endpoint** of the **Replication
  Group.** This way, in the event of a failover, your app will be resolving a DNS record that automatically gets
  updated to the latest primary node.

- To reduce the load on your primary redis node, one option is to direct reads to any of the **Read Endpoints** of the
  nodes in the Replication Group, however you now risk reading a slightly out-of-date copy of the data in the event
  that you read from a node before the primary's latest data has synced to it.

This module outputs [Terraform output variables](https://www.terraform.io/intro/getting-started/outputs.html) that
contain the address of the primary endpoint and read endpoints. You can programmatically extract these variables in
your Terraform templates and pass them to other resources (e.g. as environment variables in an EC2 Instance). You'll
also see the variables at the end of each `terraform apply` call or if you run `terraform output`.

### How do you scale the Redis Replication Group?

There are two ways to scale the Redis Replication Group:

- **Vertical:** You can increase the instance type of the individual redis nodes (individual nodes are known as "cache
  clusters") using the `instance_type` parameter (see
  [here](https://aws.amazon.com/elasticache/details/#Available_Cache_Node_Types) for valid values).

- **Horizontal:** You can add up to 6 total nodes (6 redis cache clusters) to a Replication Group. There is always a
  primary cache cluster where all writes take place, but you can reduce load on this primary cache cluster by
  offloading reads to the non-primary nodes, which are known as Read Replicas.

For more info on both methods, see [Scaling Redis Replication
Groups](http://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/Scaling.RedisReplGrps.html).

### ElastiCache for Redis Terminology

The most confusing aspect of ElastiCache for Redis is the terminology:

- **Cluster:** In ElastiCache, each Redis node is labelled its own "cluster", despite that there is only a single node
  in the cluster. This is an artifact of how ElastiCache implements memcached clusters, which can support multiple
  nodes.

- **Replication Group:** The Redis "read/write primary cluster" (the master node) will asynchronously copy its data
  to all other Redis clusters (individual nodes) in the same Replication Group. In a sense, the "real" redis cluster
  in ElastiCache is actually the Replication Group.

- **Primary Cluster:** In a Replication Group, the primary cluster is where all write traffic is directed. Initially,
  you can direct all your reads here as well.

- **Read Replicas:** In a Replication Group, data written to the primary cluster is asynchronously copied to each
  Read Replica. These are useful both to reduce the read load on the primary cluster and to faciliate fast failover.

- **Multi-AZ:** Using the Multi-AZ option means that if the primary cluster fails, ElastiCache will auto failover to
  a Read Replica. Without this option, the primary cluster node will simply replace itself, which will take longer
  and may result in additional data loss.

### Common Gotcha's

- ElastiCache Redis doesn't support sharding (however ElastiCache memcached does) so make sure your app can tolerate a
  small amount of data loss in the event that the primary node fails to sync some of its data to a read replica before
  failing.

- In the event of a Redis failover, *you will experience a small period of time during which writes are not accepted*,
  so make sure your app can handle this gracefully. In fact, consider simulating Redis failovers on a regular basis or
  with automated testing to validate that your app can handle it correctly.

- Test that your app does not cache the DNS value of the Primary Endpoint! Java, in particular, has undesirable defaults
  around DNS caching in many cases. If your code does not honor the TTL property of the Primary Endpoint's DNS record,
  then your app may fail to reach the new primary node in the event of a failure.

- The only way to add more storage space to a Redis Cluster (individual node) is to scale up to a larger node type.

- Our use of CloudFormation to make up for Terraform's lack of support for Replication Groups means we lose the
  [apply_immediately](https://www.terraform.io/docs/providers/aws/r/elasticache_cluster.html#apply_immediately)
  property. As a result, any change to the *engine version* or *node type* will be applied during the next maintenance
  window. Other modifications, such as changing the maintenance window, are applied immediately by default.

## How do you use this module?

See the [redis example](/examples/redis) for an example.

