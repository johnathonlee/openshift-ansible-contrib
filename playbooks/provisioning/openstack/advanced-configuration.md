## Dependencies for localhost (ansible control/admin node)

* [Ansible 2.3](https://pypi.python.org/pypi/ansible)
* [Ansible-galaxy](https://pypi.python.org/pypi/ansible-galaxy-local-deps)
* [jinja2](http://jinja.pocoo.org/docs/2.9/)
* [shade](https://pypi.python.org/pypi/shade)
* python-jmespath / [jmespath](https://pypi.python.org/pypi/jmespath)
* python-dns / [dnspython](https://pypi.python.org/pypi/dnspython)
* Become (sudo) is not required.

**NOTE**: You can use a Docker image with all dependencies set up.
Find more in the [Deployment section](#deployment).

### Optional Dependencies for localhost
**Note**: When using rhel images, `rhel-7-server-openstack-10-rpms` repository is required in order to install these packages.

* `python-openstackclient`
* `python-heatclient`

## Dependencies for OpenStack hosted cluster nodes (servers)

There are no additional dependencies for the cluster nodes. Required
configuration steps are done by Heat given a specific user data config
that normally should not be changed.

## Required galaxy modules

In order to pull in external dependencies for DNS configuration steps,
the following commads need to be executed:

    ansible-galaxy install \
      -r openshift-ansible-contrib/playbooks/provisioning/openstack/galaxy-requirements.yaml \
      -p openshift-ansible-contrib/roles

Alternatively you can install directly from github:

    ansible-galaxy install git+https://github.com/redhat-cop/infra-ansible,master \
      -p openshift-ansible-contrib/roles

Notes:
* This assumes we're in the directory that contains the clonned
openshift-ansible-contrib repo in its root path.
* When trying to install a different version, the previous one must be removed first
(`infra-ansible` directory from [roles](https://github.com/openshift/openshift-ansible-contrib/tree/master/roles)).
Otherwise, even if there are differences between the two versions, installation of the newer version is skipped.


## Accessing the OpenShift Cluster

### Use the Cluster DNS

In addition to the OpenShift nodes, we created a DNS server with all
the necessary entries. We will configure your *Ansible host* to use
this new DNS and talk to the deployed OpenShift.

First, get the DNS IP address:

```bash
$ openstack server show dns-0.openshift.example.com --format value --column addresses
openshift-ansible-openshift.example.com-net=192.168.99.11, 10.40.128.129
```

Note the floating IP address (it's `10.40.128.129` in this case) -- if
you're not sure, try pinging them both -- it's the one that responds
to pings.

Next, edit your `/etc/resolv.conf` as root and put `nameserver DNS_IP` as your
**first entry**.

If your `/etc/resolv.conf` currently looks like this:

```
; generated by /usr/sbin/dhclient-script
search openstacklocal
nameserver 192.168.0.3
nameserver 192.168.0.2
```

Change it to this:

```
; generated by /usr/sbin/dhclient-script
search openstacklocal
nameserver 10.40.128.129
nameserver 192.168.0.3
nameserver 192.168.0.2
```

### Get the `oc` Client

**NOTE**: You can skip this section if you're using the Docker image
-- it already has the `oc` binary.

You need to download the OpenShift command line client (called `oc`).
You can download and extract `openshift-origin-client-tools` from the
OpenShift release page:

https://github.com/openshift/origin/releases/latest/

Or you can now copy it from the master node:

    $ ansible --private-key ~/.ssh/openshift -i inventory masters[0] -m fetch -a "src=/bin/oc dest=oc"

Either way, find the `oc` binary and put it in your `PATH`.


### Logging in Using the Command Line


```bash
oc login --insecure-skip-tls-verify=true https://console.openshift.example.com:8443 -u user -p password
oc new-project test
oc new-app --template=cakephp-mysql-example
oc status -v
curl http://cakephp-mysql-example-test.apps.openshift.example.com
```

This will trigger an image build. You can run `oc logs -f
bc/cakephp-mysql-example` to follow its progress.

Wait until the build has finished and both pods are deployed and running:

```
$ oc status -v
In project test on server https://console.openshift.example.com:8443

http://cakephp-mysql-example-test.apps.openshift.example.com (svc/cakephp-mysql-example)
  dc/cakephp-mysql-example deploys istag/cakephp-mysql-example:latest <-
    bc/cakephp-mysql-example source builds https://github.com/openshift/cakephp-ex.git on openshift/php:7.0
    deployment #1 deployed about a minute ago - 1 pod

svc/mysql - 172.30.144.36:3306
  dc/mysql deploys openshift/mysql:5.7
    deployment #1 deployed 3 minutes ago - 1 pod

Info:
  * pod/cakephp-mysql-example-1-build has no liveness probe to verify pods are still running.
    try: oc set probe pod/cakephp-mysql-example-1-build --liveness ...
View details with 'oc describe <resource>/<name>' or list everything with 'oc get all'.

```

You can now look at the deployed app using its route:

```
$ curl http://cakephp-mysql-example-test.apps.openshift.example.com
```

Its `title` should say: "Welcome to OpenShift".


### Accessing the UI

You can also access the OpenShift cluster with a web browser by going to:

https://console.openshift.example.com:8443

Note that for this to work, the OpenShift nodes must be accessible
from your computer and it's DNS configuration must use the cruster's
DNS.


## Removing the OpenShift Cluster

Everything in the cluster is contained within a Heat stack. To
completely remove the cluster and all the related OpenStack resources,
run this command:

```bash
openstack stack delete --wait --yes openshift.example.com
```


## DNS configuration variables

Pay special attention to the values in the first paragraph -- these
will depend on your OpenStack environment.

Note that the provsisioning playbooks update the original Neutron subnet
created with the Heat stack to point to the configured DNS servers.
So the provisioned cluster nodes will start using those natively as
default nameservers. Technically, this allows to deploy OpenShift clusters
without dnsmasq proxies.

The `env_id` and `public_dns_domain` will form the cluster's DNS domain all
your servers will be under. With the default values, this will be
`openshift.example.com`. For workloads, the default subdomain is 'apps'.
That sudomain can be set as well by the `openshift_app_domain` variable in
the inventory.

The `openstack_<role name>_hostname` is a set of variables used for customising
hostnames of servers with a given role. When such a variable stays commented,
default hostname (usually the role name) is used.

The `public_dns_nameservers` is a list of DNS servers accessible from all
the created Nova servers. These will be serving as your DNS forwarders for
external FQDNs that do not belong to the cluster's DNS domain and its subdomains.
If you're unsure what to put in here, you can try the google or opendns servers,
but note that some organizations may be blocking them.

The `openshift_use_dnsmasq` controls either dnsmasq is deployed or not.
By default, dnsmasq is deployed and comes as the hosts' /etc/resolv.conf file
first nameserver entry that points to the local host instance of the dnsmasq
daemon that in turn proxies DNS requests to the authoritative DNS server.
When Network Manager is enabled for provisioned cluster nodes, which is
normally the case, you should not change the defaults and always deploy dnsmasq.

`external_nsupdate_keys` describes an external authoritative DNS server(s)
processing dynamic records updates in the public and private cluster views:

    external_nsupdate_keys:
      public:
        key_secret: <some nsupdate key>
        key_algorithm: 'hmac-md5'
        key_name: 'update-key'
        server: <public DNS server IP>
      private:
        key_secret: <some nsupdate key 2>
        key_algorithm: 'hmac-sha256'
        server: <public or private DNS server IP>

Here, for the public view section, we specified another key algorithm and
optional `key_name`, which normally defaults to the cluster's DNS domain.
This just illustrates a compatibility mode with a DNS service deployed
by OpenShift on OSP10 reference architecture, and used in a mixed mode with
another external DNS server.

Another example defines an external DNS server for the public view
additionally to the in-stack DNS server used for the private view only:

    external_nsupdate_keys:
      public:
        key_secret: <some nsupdate key>
        key_algorithm: 'hmac-sha256'
        server: <public DNS server IP>

Here, updates matching the public view will be hitting the given public
server IP. While updates matching the private view will be sent to the
auto evaluated in-stack DNS server's **public** IP.

Note, for the in-stack DNS server, private view updates may be sent only
via the public IP of the server. You can not send updates via the private
IP yet. This forces the in-stack private server to have a floating IP.
See also the [security notes](#security-notes)

## Other configuration variables

`openstack_ssh_key` is a Nova keypair - you can see your keypairs with
`openstack keypair list`. This guide assumes that its corresponding private
key is `~/.ssh/openshift`, stored on the ansible admin (control) node.

`openstack_default_image_name` is the default name of the Glance image the
servers will use. You can see your images with `openstack image list`.
In order to set a different image for a role, uncomment the line with the
corresponding variable (e.g. `openstack_lb_image_name` for load balancer) and
set its value to another available image name. `openstack_default_image_name`
must stay defined as it is used as a default value for the rest of the roles.

`openstack_default_flavor` is the default Nova flavor the servers will use.
You can see your flavors with `openstack flavor list`.
In order to set a different flavor for a role, uncomment the line with the
corresponding variable (e.g. `openstack_lb_flavor` for load balancer) and
set its value to another available flavor. `openstack_default_flavor` must
stay defined as it is used as a default value for the rest of the roles.

`openstack_external_network_name` is the name of the Neutron network
providing external connectivity. It is often called `public`,
`external` or `ext-net`. You can see your networks with `openstack
network list`.

`openstack_private_network_name` is the name of the private Neutron network
providing admin/control access for ansible. It can be merged with other
cluster networks, there are no special requirements for networking.

The `openstack_num_masters`, `openstack_num_infra` and
`openstack_num_nodes` values specify the number of Master, Infra and
App nodes to create.

The `openshift_cluster_node_labels` defines custom labels for your openshift
cluster node groups. It currently supports app and infra node groups.
The default value of this variable sets `region: primary` to app nodes and
`region: infra` to infra nodes.
An example of setting a customised label:
```
openshift_cluster_node_labels:
  app:
    mylabel: myvalue
```

The `openstack_nodes_to_remove` allows you to specify the numerical indexes
of App nodes that should be removed; for example, ['0', '2'],

The `docker_volume_size` is the default Docker volume size the servers will use.
In order to set a different volume size for a role,
uncomment the line with the corresponding variable (e. g. `docker_master_volume_size`
for master) and change its value. `docker_volume_size` must stay defined as it is
used as a default value for some of the servers (master, infra, app node).
The rest of the roles (etcd, load balancer, dns) have their defaults hard-coded.

**Note**: If the `ephemeral_volumes` is set to `true`, the `*_volume_size` variables
will be ignored and the deployment will not create any cinder volumes.

The `openstack_flat_secgrp`, controls Neutron security groups creation for Heat
stacks. Set it to true, if you experience issues with sec group rules
quotas. It trades security for number of rules, by sharing the same set
of firewall rules for master, node, etcd and infra nodes.

The `required_packages` variable also provides a list of the additional
prerequisite packages to be installed before to deploy an OpenShift cluster.
Those are ignored though, if the `manage_packages: False`.

The `openstack_inventory` controls either a static inventory will be created after the
cluster nodes provisioned on OpenStack cloud. Note, the fully dynamic inventory
is yet to be supported, so the static inventory will be created anyway.

The `openstack_inventory_path` points the directory to host the generated static inventory.
It should point to the copied example inventory directory, otherwise ti creates
a new one for you.

## Multi-master configuration

Please refer to the official documentation for the
[multi-master setup](https://docs.openshift.com/container-platform/3.6/install_config/install/advanced_install.html#multiple-masters)
and define the corresponding [inventory
variables](https://docs.openshift.com/container-platform/3.6/install_config/install/advanced_install.html#configuring-cluster-variables)
in `inventory/group_vars/OSEv3.yml`. For example, given a load balancer node
under the ansible group named `ext_lb`:

    openshift_master_cluster_method: native
    openshift_master_cluster_hostname: "{{ groups.ext_lb.0 }}"
    openshift_master_cluster_public_hostname: "{{ groups.ext_lb.0 }}"

## Provider Network

Normally, the playbooks create a new Neutron network and subnet and attach
floating IP addresses to each node. If you have a provider network set up, this
is all unnecessary as you can just access servers that are placed in the
provider network directly.

To use a provider network, set its name in `openstack_provider_network_name` in
`inventory/group_vars/all.yml`.

If you set the provider network name, the `openstack_external_network_name` and
`openstack_private_network_name` fields will be ignored.

**NOTE**: this will not update the nodes' DNS, so running openshift-ansible
right after provisioning will fail (unless you're using an external DNS server
your provider network knows about). You must make sure your nodes are able to
resolve each other by name.

## Security notes

Configure required `*_ingress_cidr` variables to restrict public access
to provisioned servers from your laptop (a /32 notation should be used)
or your trusted network. The most important is the `node_ingress_cidr`
that restricts public access to the deployed DNS server and cluster
nodes' ephemeral ports range.

Note, the command ``curl https://api.ipify.org`` helps fiding an external
IP address of your box (the ansible admin node).

There is also the `manage_packages` variable (defaults to True) you
may want to turn off in order to speed up the provisioning tasks. This may
be the case for development environments. When turned off, the servers will
be provisioned omitting the ``yum update`` command. This brings security
implications though, and is not recommended for production deployments.

### DNS servers security options

Aside from `node_ingress_cidr` restricting public access to in-stack DNS
servers, there are following (bind/named specific) DNS security
options available:

    named_public_recursion: 'no'
    named_private_recursion: 'yes'

External DNS servers, which is not included in the 'dns' hosts group,
are not managed. It is up to you to configure such ones.

## Configure the OpenShift parameters

Finally, you need to update the DNS entry in
`inventory/group_vars/OSEv3.yml` (look at
`openshift_master_default_subdomain`).

In addition, this is the place where you can customise your OpenShift
installation for example by specifying the authentication.

The full list of options is available in this sample inventory:

https://github.com/openshift/openshift-ansible/blob/master/inventory/byo/hosts.ose.example

Note, that in order to deploy OpenShift origin, you should update the following
variables for the `inventory/group_vars/OSEv3.yml`, `all.yml`:

    deployment_type: origin
    openshift_deployment_type: "{{ deployment_type }}"


## Setting a custom entrypoint

In order to set a custom entrypoint, update `openshift_master_cluster_public_hostname`

    openshift_master_cluster_public_hostname: api.openshift.example.com

Note than an empty hostname does not work, so if your domain is `openshift.example.com`,
you cannot set this value to simply `openshift.example.com`.

## Creating and using a Cinder volume for the OpenShift registry

You can optionally have the playbooks create a Cinder volume and set
it up as the OpenShift hosted registry.

To do that you need specify the desired Cinder volume name and size in
Gigabytes in `inventory/group_vars/all.yml`:

    cinder_hosted_registry_name: cinder-registry
    cinder_hosted_registry_size_gb: 10

With this, the playbooks will create the volume and set up its
filesystem. If there is an existing volume of the same name, we will
use it but keep the existing data on it.

To use the volume for the registry, you must first configure it with
the OpenStack credentials by putting the following to `OSEv3.yml`:

    openshift_cloudprovider_openstack_username: "{{ lookup('env','OS_USERNAME') }}"
    openshift_cloudprovider_openstack_password: "{{ lookup('env','OS_PASSWORD') }}"
    openshift_cloudprovider_openstack_auth_url: "{{ lookup('env','OS_AUTH_URL') }}"
    openshift_cloudprovider_openstack_tenant_name: "{{ lookup('env','OS_TENANT_NAME') }}"

This will use the credentials from your shell environment. If you want
to enter them explicitly, you can. You can also use credentials
different from the provisioning ones (say for quota or access control
reasons).

**NOTE**: If you're testing this on (DevStack)[devstack], you must
explicitly set your Keystone API version to v2 (e.g.
`OS_AUTH_URL=http://10.34.37.47/identity/v2.0`) instead of the default
value provided by `openrc`. You may also encounter the following issue
with Cinder:

https://github.com/kubernetes/kubernetes/issues/50461

You can read the (OpenShift documentation on configuring
OpenStack)[openstack] for more information.

[devstack]: https://docs.openstack.org/devstack/latest/
[openstack]: https://docs.openshift.org/latest/install_config/configuring_openstack.html


Next, we need to instruct OpenShift to use the Cinder volume for it's
registry. Again in `OSEv3.yml`:

    #openshift_hosted_registry_storage_kind: openstack
    #openshift_hosted_registry_storage_access_modes: ['ReadWriteOnce']
    #openshift_hosted_registry_storage_openstack_filesystem: xfs

The filesystem value here will be used in the initial formatting of
the volume.

If you're using the dynamic inventory, you must uncomment these two values as
well:

    #openshift_hosted_registry_storage_openstack_volumeID: "{{ lookup('os_cinder', cinder_hosted_registry_name).id }}"
    #openshift_hosted_registry_storage_volume_size: "{{ cinder_hosted_registry_size_gb }}Gi"

But note that they use the `os_cinder` lookup plugin we provide, so you must
tell Ansible where to find it either in `ansible.cfg` (the one we provide is
configured properly) or by exporting the
`ANSIBLE_LOOKUP_PLUGINS=openshift-ansible-contrib/lookup_plugins` environment
variable.



## Use an existing Cinder volume for the OpenShift registry

You can also use a pre-existing Cinder volume for the storage of your
OpenShift registry.

To do that, you need to have a Cinder volume. You can create one by
running:

    openstack volume create --size <volume size in gb> <volume name>

The volume needs to have a file system created before you put it to
use.

As with the automatically-created volume, you have to set up the
OpenStack credentials in `inventory/group_vars/OSEv3.yml` as well as
registry values:

    #openshift_hosted_registry_storage_kind: openstack
    #openshift_hosted_registry_storage_access_modes: ['ReadWriteOnce']
    #openshift_hosted_registry_storage_openstack_filesystem: xfs
    #openshift_hosted_registry_storage_openstack_volumeID: e0ba2d73-d2f9-4514-a3b2-a0ced507fa05
    #openshift_hosted_registry_storage_volume_size: 10Gi

Note the `openshift_hosted_registry_storage_openstack_volumeID` and
`openshift_hosted_registry_storage_volume_size` values: these need to
be added in addition to the previous variables.

The **Cinder volume ID**, **filesystem** and **volume size** variables
must correspond to the values in your volume. The volume ID must be
the **UUID** of the Cinder volume, *not its name*.

We can do formate the volume for you if you ask for it in
`inventory/group_vars/all.yml`:

    prepare_and_format_registry_volume: true

**NOTE:** doing so **will destroy any data that's currently on the volume**!

You can also run the registry setup playbook directly:

   ansible-playbook -i inventory playbooks/provisioning/openstack/prepare-and-format-cinder-volume.yaml

(the provisioning phase must be completed, first)



## Configure static inventory and access via a bastion node

Example inventory variables:

    openstack_use_bastion: true
    bastion_ingress_cidr: "{{openstack_subnet_prefix}}.0/24"
    openstack_private_ssh_key: ~/.ssh/openshift
    openstack_inventory: static
    openstack_inventory_path: ../../../../inventory
    openstack_ssh_config_path: /tmp/ssh.config.openshift.ansible.openshift.example.com

The `openstack_subnet_prefix` is the openstack private network for your cluster.
And the `bastion_ingress_cidr` defines accepted range for SSH connections to nodes
additionally to the `ssh_ingress_cidr`` (see the security notes above).

The SSH config will be stored on the ansible control node by the
gitven path. Ansible uses it automatically. To access the cluster nodes with
that ssh config, use the `-F` prefix, f.e.:

    ssh -F /tmp/ssh.config.openshift.ansible.openshift.example.com master-0.openshift.example.com echo OK

Note, relative paths will not work for the `openstack_ssh_config_path`, but it
works for the `openstack_private_ssh_key` and `openstack_inventory_path`. In this
guide, the latter points to the current directory, where you run ansible commands
from.

To verify nodes connectivity, use the command:

    ansible -v -i inventory/hosts -m ping all

If something is broken, double-check the inventory variables, paths and the
generated `<openstack_inventory_path>/hosts` and `openstack_ssh_config_path` files.

The `inventory: dynamic` can be used instead to access cluster nodes directly via
floating IPs. In this mode you can not use a bastion node and should specify
the dynamic inventory file in your ansible commands , like `-i openstack.py`.

## Using Docker on the Ansible host

If you don't want to worry about the dependencies, you can use the
[OpenStack Control Host image][control-host-image].

[control-host-image]: https://hub.docker.com/r/redhatcop/control-host-openstack/

It has all the dependencies installed, but you'll need to map your
code and credentials to it. Assuming your SSH keys live in `~/.ssh`
and everything else is in your current directory (i.e. `ansible.cfg`,
`keystonerc`, `inventory`, `openshift-ansible`,
`openshift-ansible-contrib`), this is how you run the deployment:

    sudo docker run -it -v ~/.ssh:/mnt/.ssh:Z \
        -v $PWD:/root/openshift:Z \
        -v $PWD/keystonerc:/root/.config/openstack/keystonerc.sh:Z \
        redhatcop/control-host-openstack bash

(feel free to replace `$PWD` with an actual path to your inventory and
checkouts, but note that relative paths don't work)

The first run may take a few minutes while the image is being
downloaded. After that, you'll be inside the container and you can run
the playbooks:

    cd openshift
    ansible-playbook openshift-ansible-contrib/playbooks/provisioning/openstack/provision.yaml


### Run the playbook

Assuming your OpenStack (Keystone) credentials are in the `keystonerc`
this is how you stat the provisioning process from your ansible control node:

    . keystonerc
    ansible-playbook openshift-ansible-contrib/playbooks/provisioning/openstack/provision.yaml

Note, here you start with an empty inventory. The static inventory will be populated
with data so you can omit providing additional arguments for future ansible commands.

If bastion enabled, the generates SSH config must be applied for ansible.
Otherwise, it is auto included by the previous step. In order to execute it
as a separate playbook, use the following command:

    ansible-playbook openshift-ansible-contrib/playbooks/provisioning/openstack/post-provision-openstack.yml

The first infra node then becomes a bastion node as well and proxies access
for future ansible commands. The post-provision step also configures Satellite,
if requested, and DNS server, and ensures other OpenShift requirements to be met.

## Running Custom Post-Provision Actions

A custom playbook can be run like this:

```
ansible-playbook --private-key ~/.ssh/openshift -i inventory/ openshift-ansible-contrib/playbooks/provisioning/openstack/custom-actions/custom-playbook.yml
```

If you'd like to limit the run to one particular host, you can do so as follows:

```
ansible-playbook --private-key ~/.ssh/openshift -i inventory/ openshift-ansible-contrib/playbooks/provisioning/openstack/custom-actions/custom-playbook.yml -l app-node-0.openshift.example.com
```

You can also create your own custom playbook. Here's one example that adds additional YUM repositories:

```
---
- hosts: app
  tasks:

  # enable EPL
  - name: Add repository
    yum_repository:
      name: epel
      description: EPEL YUM repo
      baseurl: https://download.fedoraproject.org/pub/epel/$releasever/$basearch/
```

This example runs against app nodes. The list of options include:

  - cluster_hosts (all hosts: app, infra, masters, dns, lb)
  - OSEv3 (app, infra, masters)
  - app
  - dns
  - masters
  - infra_hosts

Please consider contributing your custom playbook back to openshift-ansible-contrib!

A library of custom post-provision actions exists in `openshift-ansible-contrib/playbooks/provisioning/openstack/custom-actions`. Playbooks include:

### add-yum-repos.yml

[add-yum-repos.yml](https://github.com/openshift/openshift-ansible-contrib/blob/master/playbooks/provisioning/openstack/custom-actions/add-yum-repos.yml) adds a list of custom yum repositories to every node in the cluster.

## Install OpenShift

Once it succeeds, you can install openshift by running:

    ansible-playbook openshift-ansible/playbooks/byo/config.yml

## Access UI

OpenShift UI may be accessed via the 1st master node FQDN, port 8443.

When using a bastion, you may want to make an SSH tunnel from your control node
to access UI on the `https://localhost:8443`, with this inventory variable:

   openshift_ui_ssh_tunnel: True

Note, this requires sudo rights on the ansible control node and an absolute path
for the `openstack_private_ssh_key`. You should also update the control node's
`/etc/hosts`:

    127.0.0.1 master-0.openshift.example.com

In order to access UI, the ssh-tunnel service will be created and started on the
control node. Make sure to remove these changes and the service manually, when not
needed anymore.

## Scale Deployment up/down

### Scaling up

One can scale up the number of application nodes by executing the ansible playbook
`openshift-ansible-contrib/playbooks/provisioning/openstack/scale-up.yaml`.
This process can be done even if there is currently no deployment available.
The `increment_by` variable is used to specify by how much the deployment should
be scaled up (if none exists, it serves as a target number of application nodes).
The path to `openshift-ansible` directory can be customised by the `openshift_ansible_dir`
variable. Its value must be an absolute path to `openshift-ansible` and it cannot
contain the '/' symbol at the end.

Usage:

```
ansible-playbook -i <path to inventory> openshift-ansible-contrib/playbooks/provisioning/openstack/scale-up.yaml` [-e increment_by=<number>] [-e openshift_ansible_dir=<path to openshift-ansible>]
```

Note: This playbook works only without a bastion node (`openstack_use_bastion: False`).