# Supported configuration parameters
The playbook in the [provision-cassandra.yml](../provision-cassandra.yml) file in this repository pulls in a set of default values for many of the configuration parameters that are needed to deploy Cassandra from the [vars/cassandra.yml](../vars/cassandra.yml) and [defaults/main.yml](../defaults/main.yml) files. The parameters defined in these files define a reasonable set of defaults for a fairly generic Cassandra deployment, either to a single node or a cluster, including defaults for the URL that the Cassandra distribution should be downloaded from, the directory the distribution should be unpacked into, and the packages that must be installed on the node before the `cassandra` service can be started.

In addition to the defaults defined in the [vars/cassandra.yml](../vars/cassandra.yml) and [defaults/main.yml](../defaults/main.yml) files, there are a large number of parameters that can be used to either control the deployment of Cassandra to the nodes that will make up a cluster during an `ansible-playbook` run or to configure those Cassandra nodes once the installation is complete. In this section, we summarize all of these options, breaking them out into:

* parameters used to control the `ansible-playbook` run
* parameters used during the deployment process itself, and
* parameters used to configure our Cassandra nodes once Cassandra has been installed locally.

Each of these sets of parameters are described in their own section, below.

## Parameters used to control the playbook run
The following parameters can be used to control the `ansible-playbook` run itself, defining things like how Ansible should connect to the nodes involved in the playbook run, which nodes should be targeted, where the Cassandra distribution should be downloaded from, which packages must be installed during the deployment process, and where those packages should be obtained from:

* **`ansible_ssh_private_key_file`**: the location of the private key that should be used when connecting to the target nodes via SSH; this parameter is useful when there is one private key that is used to connect to all of the target nodes in a given playbook run
* **`ansible_user`**: the username that should be used when connecting to the target nodes via SSH; is useful if the same username is used when connecting to all of the target nodes in a given playbook run
* **`cassandra_url`**: the URL that the Cassandra distribution should be downloaded from
* **`cassandra_version`**: the version of Cassandra that should be downloaded; used to switch versions when the distribution is downloaded using the default `cassandra_url`, which is defined in the [defaults/main.yml](../defaults/main.yml) file
* **`cloud`**: if the inventory is being managed dynamically, this parameter is used to indicate the type of target cloud for the deployment (either `aws` or `osp`); this controls how the [build-app-host-groups](../common-roles/build-app-host-groups) common role retrieves the list of target nodes for the deployment
* **`local_vars_file`**: used to define the location of a *local variables file* (see the discussion of this topic, below); this file is a YAML file containing definitions for any of the configuration parameters that are described in this section and is more than likely a file that will be created to manage the process of creating a specific cluster (or adding nodes to that cluster). Storing the settings for a given cluster in such a file makes it easy to guarantee that all of the nodes in that cluster are configured consistently
* **`private_key_path`**: used to define the directory where the private keys are maintained when the inventory for the playbook run is being managed dynamically; in these cases, the scripts used to retrieve the dynamic inventory information will return the names of the keys that should be used to access each node, and the playbook will search the directory specified by this parameter to find the corresponding key files. If this value is not specified then the current working directory will be searched for those keys by default
* **`proxy_env`**: a hash map that is used to define the proxy settings to use for downloading distribution files and installing packages; supports the `http_proxy`, `no_proxy`, `proxy_username`, and `proxy_password` fields as part of this hash map
* **`reset_proxy_settings`**: used to reset any HTTP/YUM proxy settings that may have been made in a previous playbook run back to the defaults (no proxy); this is useful when a proxy was incorrectly set in a previous playbook run and the user wants to return to a "no-proxy" setup in the current playbook run
* **`yum_repo_url`**: used to set the URL for a local YUM mirror. This parameter is only used for CentOS-based deployments; when deploying Cassandra to RHEL-based nodes this parameter is silently ignored and the RHEL package repositories defined locally on the node will be used for any packages installed during the deployment process

## Parameters used during the deployment process
These parameters are used to control the deployment process itself, defining things like where to unpack the distribution into, whether or not Cassandra should be started when the deployment process is complete, and what user/group the should be used when running Cassandra locally.

* **`cassandra_dir`**: the directory that the Cassandra distribution should be unpacked into; defaults to the `/opt/apache-cassandra` directory. If necessary, this directory will be created as part of the playbook run
* **`cassandra_package_list`**: the list of packages that should be installed on the Cassandra nodes; typically this parameter is left unchanged from the default (which installs the OpenJDK packages needed to run Cassandra), but if it is modified the default, OpenJDK packages must be included as part of this list or an error will result when attempting to start the `cassandra` service
* **`cassandra_group`**: the name of the user group under which Cassandra should be installed and run; defaults to `cassandra`
* **`cassandra_user`**: the username under which Cassandra should be installed and run; defaults to `cassandra`
* **`start_cassandra`**: this parameter is used control whether or not the `cassandra` service should be started when the playbook is complete; it defaults to `false` as a safety measure, since the `cassandra` service should not be started when adding new nodes to "grow" an existing cluster that contains data. In that case, each node will have to be brought up separately and the cluster will have to be allowed to rebalance its the data/workload across the new cluster topology before another node can be added. In initial deployments (where there is no data in the database yet) this parameter can be set to `true`, but otherwise it should be set to `false`.

## Parameters used to configure the Cassandra nodes
These parameters are used configure the Cassandra nodes themselves during a playbook run, defining things like the interfaces that Cassandra should be listening on for requests, the directory where Cassandra should store its data, and the list of seed nodes for the cluster.

* **`data_iface`**: the name of the interface that the Cassandra nodes should use when talking with each other. This interface typically corresponds to a private or management network, with no customer access. An interface of this name must exist for the playbook to run successfully, and if unspecified a value of `eth0` is assumed
* **`api_iface`**: the name of the interface that the Cassandra nodes should use when handling user requests. This network corresponds to a public network since customers will use this interface to access the data in the Cassandra cluster. An interface of this name must exist for the playbook to run successfully, and if unspecified a value of `eth0` is assumed
* **`iface_description_array`**: this parameter can be used in place of the `data_iface` and `api_iface` parameters described above, and it provides users with the ability to specify a description of these two interfaces rather than identifying them by name (more on this, below)
* **`cassandra_data_dir`**: the name of the directory that Cassandra should use to store its data; defaults to `/data` if unspecified. If necessary, this directory will be created as part of the playbook run
* **`auto_bootstrap`**: used to set the parameter of the same name in the `cassandra.yaml` file; by default Cassandra assumes that this parameter is true if it is not specified in the `cassandra.yaml` file, but it may be set to false when initializing a fresh cluster *without data*. It is critical that this be set to `true` when adding new nodes to an existing cluster that includes data
* **`cluster_name_suffix`**: in the dynamic inventory use case this parameter can be used to specify a string that should be added a suffix to the default cluster name that is generated by the playbook. By default, the playbook will create a cluster name by concatenating the tenant and project values that were passed into the playbook, and this value can be used to make the cluster name for a given deployment unique. If left unspecified this suffix defaults to the empty string and the all of the clusters created for a given tenant and project will all have the same name
* **`cassandra_seed_nodes`**: this parameter is used to pass in the list of seed nodes for the cluster; it is required for any multi-node (clustered) Cassandra deployment and should be consistently defined for all nodes in the cluster

In addition to these parameters, Cassandra supports an extremely large set of configuration parameters that *may* be set by the user if they wish (reasonable defaults are assumed for all of these parameters in the playbook run if they are not specified). Rather than listing all of these parameters here, we will briefly describe the strategy that is used by the playbook when assigning values to these variables here.

Briefly, for any parameter defined in the `cassandra.yaml` configuration file that is included in the Cassandra distribution (the `rpc_port` for example), the user can define a value for that parameter using a variable of the same name but with a `cassandra_` prefix. So, to change the `rpc_port` from the default value (9160), you would simply assign your desired value to the `cassandra_rpc_port` variable (passing that variable into the `ansible-playbook` run using one of the strategies outline in the discussion in the *Controlling the configuration* of the [README.md](../README.md) file), and that value would override the default value in the `cassandra.yaml` file for all of the nodes targeted by your playbook run.

In the current (v3.10) Cassandra release there are more than 130 of these configuration parameters that users may wish to customize in the `cassandra.yaml` file, most of which will never be set to anything but their default values. That said, the strategy followed here does give users the ability to set any of these parameters that they wish to set in their own deployments without requiring either an excessive amount of work on our part or an excessive amount of work on the part of users who do not wish to set values for those parameters.

## Interface names vs. interface descriptions
For some operating systems on some platforms, it can be difficult (if not impossible) to determine the names of the interfaces that should be passed into the playbook using the `data_iface` and `api_iface` parameters that we described, above. In those situations, the playbook in this repository provides an alternative; specifying those interfaces using the `iface_description_array` parameter instead.

Put quite simply, the `iface_description_array` lets you specify a description for each of the networks that you are interested in, then retrieve the names of those networks on each machine in a variable that can be used elsewhere in the playbook. To accomplish this, the `iface_description_array` is defined as an array of hashes (one per interface), each of which include the following fields:

* **`type`**: the type of description being provided, currently only the `cidr` type is supported
* **`val`**: a value describing the network in question; since only `cidr` descriptions are currently supported, a CIDR value that looks something like `192.168.34.0/24` should be used for this field
* **`as_var`**: the name of the variable that you would like the interface name returned as

With these values in hand, the playbook will search the available networks on each machine and return a list of the interface names for each network that was described in the `iface_description_array` as the value of the fact named in the `as_var` field for that network's entry. For example, given this description:

```json
    iface_description_array: [
        { as_var: 'data_iface', type: 'cidr', val: '192.168.34.0/24' },
        { as_var: 'api_iface', type: 'cidr', val: '192.168.44.0/24' },
    ]
```

the playbook will return the name of the network that matches the CIDR `192.168.34.0/24` as the value of the `data_iface` fact and the name of the network that matches the CIDR `192.168.34.0/24` as the value of the `api_iface` fact. These two facts can then be used later in the playbook to correctly configure the nodes to talk to each other and listen on the proper interfaces for user requests.

It should be noted that if you choose to take this approach when constructing your `ansible-playbook` runs, a matching entry in the `iface_description_array` must be specified for both the `data_iface` and `api_iface` networks, otherwise the default value of `eth0` will be used for these facts (and the playbook run may result in nodes that are at best misconfigured; if the `eth0` network does not exist then the playbook will fail to run altogether).
