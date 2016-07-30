**Note**: This public repo contains the documentation for the private GitHub repo <https://github.com/gruntwork-io/module-cache>.
We publish the documentation publicly so it turns up in online searches, but to see the source code, you must be a Gruntwork customer.
If you're already a Gruntwork customer, the original source for this file is at: <https://github.com/gruntwork-io/module-cache/blob/master/examples/memcached/README.md>.
If you're not a customer, contact us at <info@gruntwork.io> or <http://www.gruntwork.io> for info on how to get access!

# Memcached Example

This folder contains an example of how to use the [Memcached module](/modules/memcached) to create an ElastiCache
cluster cluster that runs [Memcached](https://memcached.org/).

## How do you run this example?

To run this example, you need to:

1. Install [Terraform](https://www.terraform.io/).
1. Open up `vars.tf` and set secrets at the top of the file as environment variables and fill in any other variables in
   the file that don't have defaults. 
1. `terraform get`.
1. `terraform plan`.
1. If the plan looks good, run `terraform apply`.

*NOTE: To automatically enforce terraform best practices, for anything beyond local experimentation, we recommend using 
[terragrunt](https://github.com/gruntwork-io/terragrunt).*

When the templates are applied, Terraform will output the IP addresses of the cache nodes. 