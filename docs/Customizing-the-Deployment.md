# Customizing the deployment
As was mentioned in [this document](Deployment-Scenarios.md), there are a number of ways that the deployment process can be customized to suit your needs. In this document, we briefly discuss the files where the default values for the parameters used during the playbook run are defined, their intended purpose, and how you can override those default values in your own playbook runs.

## Files used during the playbook run
There are two files where the default values for the parameters used during a playbook run are defined; for the [storm repository](https://github.com/Datanexus/storm) those files are as follows:

### The [vars/storm.yml](../vars/storm.yml) file
This **variables file** defines a reasonable set of defaults for the variables used during all [provision-storm.yml](../provision-storm.yml) playbook runs, regardless of the target environment. Most of the variables defined in this file do not need to be modified from one playbook run to the next, although the values defined here can be overridden by redefining them in the **configuration file** that is used during that playbook run. We will discuss our recommendations for customizing the values found in this file [later in this document](#customization-options).

### The [config.yml](../config.yml) file

This file is the **default configuration file** used during a [provision-storm.yml](../provision-storm.yml) playbook run if a custom configuration file is not provided at runtime. The intent of this default configuration file is to provide values for the variables that we expect to change from one playbook run to another based on the target environment. This file is maintained under version control as part of the [storm repository](https://github.com/Datanexus/storm), and the current released version of this file looks like this:

```yaml
# (c) 2017 DataNexus Inc.  All Rights Reserved
#
# Defaults used if a configuration file is not passed into the playbook run
# using the `config_file` extra variable; deploys to an AWS environment
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
# network configuration used when creating VMs
cidr_block: '10.10.0.0/16'
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# node type and image ID to deploy; note that the image ID is optional for AWS
# deployments but mandatory for OSP deployments; if left undefined in an AWS
# deployment then then the playbook will search for an appropriate image to use
type: 't2.large'
#image: 'ami-f4533694'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
# the default data and root volume sizes
root_volume: 11
data_volume: 40
```

The variables that are included in this file are the variables that we expect to change for **any** playbook run; whereas the variables that are defined in the [vars/storm.yml](../vars/storm.yml) file represent a reasonable set of defaults that could be used for **all** playbook runs, regardless of the type of deployment that is being made.

In short, the variables defined in this default configuration file are the small set of parameters that will change from one playbook run to the next. In the file shown above, you can see that this default configuration file will use the defined `cloud`, `tenant`, `project`, and `domain` values to search for (and construct if necessary) a set of `t2.small` machines in the AWS, US West (Oregon) region for use in the playbook run. There are two additional tags that are used for this search (but not shown in this example configuration file); those two tags (the `dataflow` and `cluster` tags) are optional, and will take on a default value (`none` and `a`, respectively) during the playbook run if they are not specified at runtime (either in a configuration file or as extra variables on the command line).

## Customization options
There are three basic mechanisms for customing a given playbook run:

* Making changes to the values defined in the **variables file** (by editing the [vars/storm.yml](../vars/storm.yml) file directly)
* Making changes to the values defined in the **default configuration file** (by editing the [config.yml](../config.yml) file directly)
* Constructing your own configuration file and passing that file into the playbook run at runtime using the `config_file` extra variable

While the first two options may seem attractive at first, the files in question are maintained under version control in the main [storm repository](https://github.com/Datanexus/storm) repository. As such, edits to these files may result in merge conflicts down the line when you attempt to update your clone of the main [storm repository](https://github.com/Datanexus/storm) repository by pulling down the latest updates and bug fixes.

To avoid this problem, our recommendation is that you create a new, custom configuration file containing any parameters where you would like to override the default value (whether those variables are defined in the [vars/storm.yml](../vars/storm.yml) file or the [config.yml](../config.yml) file), then pass that custom configuration file into the playbook at runtime using the `config_file` extra variable. For example, if you wanted to deploy a set of `t2.small` machines and increase the data volume size for the Kafka instances running in those VMs to 100GB, then you might setup a custom configuration file that looks something like the following:

```yaml
$ cat config-aws-custom.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# network configuration used when creating VMs
cidr_block: '10.10.0.0/16'
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# node type and image ID to deploy; note that the image ID is optional for AWS
# deployments but mandatory for OSP deployments; if left undefined in an AWS
# deployment then then the playbook will search for an appropriate image to use
type: 't2.small'
image: 'ami-f4533694'
# the default region to search (the us-west-2 EC2 region)
region: us-west-2
# the default data and root volume sizes
root_volume: 11
data_volume: 40
# custom Storm configuration parameters (sets the max heap size for the
# various processes so that they can be started in a `t2.small` node)
nimbus_childopts: "-Xmx768m"
ui_childopts: "-Xmx512m"
worker_childopts: "-Xmx512m"
```

Note that we have redefined the parameters that are defined in the default configuration file (the [config.yml](../config.yml) file) here, since we will be replacing the default configuration file with the custom configuration file shown above. Also note that in this custom configuration file we have:

* specified the image ID to use for this deployment when creating images; if this image ID is not specified (for an AWS deployment) then the playbook will search for a suitable image to use, by specifying it here we save some time during our deployment
* increased the data volume size of the Kafka instances we're deploying from the value of 40GB that is defined in the [config.yml](../config.yml) file to a new value of 100GB
* overridden a few of the parameters from the [vars/storm.yml](,,/vars/storm.yml) file that are related to the maximum heap sizes for the JVM instances that will be used to run Storm so that those JVM instances can be started on a `t2.small` instance

Once we've constructed our custom configuration file, we could then use that file to deploy a cluster by simply passing that configuration file into our playbook via the `config_file` variable (using either a full or relative path to that file):

```bash
$ AWS_PROFILE=datanexus_west ./provision-storm.yml -e "{ \
    config_file: config-aws-custom.yml \
}"
```

Any of the variables that are set in the [vars/storm.yml](../vars/storm.yml) and [config.yml](../config.yml) files in this repository can be overridden in this manner. Just keep in mind that any variables defined in the default configuration file (the [config.yml](../config.yml) file) must always be redefined in any custom configuration file that you create (unless, of course, you don't need those variables for the deployment that you are performing).
