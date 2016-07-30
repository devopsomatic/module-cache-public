**Note**: This public repo contains the documentation for the private GitHub repo <https://github.com/gruntwork-io/module-cache>.
We publish the documentation publicly so it turns up in online searches, but to see the source code, you must be a Gruntwork customer.
If you're already a Gruntwork customer, the original source for this file is at: <https://github.com/gruntwork-io/module-cache/blob/master/modules/memcached/README.md>.
If you're not a customer, contact us at <info@gruntwork.io> or <http://www.gruntwork.io> for info on how to get access!

# Memcached Module

This module creates an ElastiCache cluster that runs [Memcached](https://memcached.org/).

## How do you use this module?

See the [memcached example](/examples/memcached) for an example. 

## How do you connect to the Memcached cluster?

This module outputs a [Terraform output variable](https://www.terraform.io/intro/getting-started/outputs.html) that
contains a comma-separated list of addresses of the Memcached nodes. You can programmatically extract this variable in 
your Terraform templates and pass it to other resources (e.g. as an environment variable in an EC2 instance). You'll 
also see the variable at the end of each `terraform apply` call or if you run `terraform output`.

## How do you scale this Memcached cluster?

* To scale vertically, increase the size of the nodes using the `instance_type` parameter (see 
  [here](https://aws.amazon.com/elasticache/details/#Available_Cache_Node_Types) for valid values). 
* To scale horizontally, increase the number of nodes using the `num_cache_nodes` parameter.  

For more info, see [Scaling Memcached](http://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/Scaling.Memcached.html).