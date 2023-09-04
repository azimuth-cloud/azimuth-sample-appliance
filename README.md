# azimuth-sample-appliance  <!-- omit in toc -->

This repository contains a sample appliance that also serves as documentation on how
to build an appliance for use with [Azimuth](https://github.com/stackhpc/azimuth)
Cluster-as-a-Service (CaaS).

The appliance uses [Terraform](https://www.terraform.io/) to provision resources (e.g.
servers, security groups, ports, volumes) and [Ansible](https://www.ansible.com/) to
configure them, taking advantage of Ansible's support for dynamic inventory to bridge
the two components.

## Contents  <!-- omit in toc -->

- [Sample appliance architecture](#sample-appliance-architecture)
- [Azimuth CaaS Operator](#azimuth-caas-operator)
- [Ansible variables](#ansible-variables)
- [Cluster metadata](#cluster-metadata)
- [Resource provisioning](#resource-provisioning)
- [Persistent state](#persistent-state)
- [Cluster patching](#cluster-patching)
- [Cluster outputs](#cluster-outputs)

## Sample appliance architecture

This sample appliance is a simple HTTP service, consisting of one or more backends with
a load-balancer in front.

The load-balancer is an [Nginx](https://nginx.org/) server that is configured to forward
traffic to the backends. The backends are also Nginx servers, and are configured to render
a single page that contains the IP of the host on the internal network. Hence when hitting
the service multiple times, the end user should see a different IP address reported each
time (when more than one backend is configured!).

<!-- When [Zenith](https://github.com/stackhpc/zenith) is enabled for the target Azimuth
deployment, the appliance exposes the load-balancer using a Zenith domain. An ephemeral
[SSH jump host](https://wiki.gentoo.org/wiki/SSH_jump_host) with a floating IP is used
to allow Ansible to access the hosts for configuration.

When Zenith is not enabled, the load-balancer is exposed using a floating IP which is
reported to the user using the cluster outputs. -->

The load-balancer is exposed using a floating IP which is reported to the user using the
cluster outputs.

## Azimuth CaaS Operator

Azimuth CaaS appliances are driven using the [Azimuth CaaS Operator](https://github.com/stackhpc/azimuth-caas-operator). 
In practice, this makes very little difference other than some small constraints on the layout of your
repository:

  * Entrypoint playbooks should be at the top level of the repository, e.g.
    [sample-appliance.yml](./sample-appliance.yml)
  * Roles should be in a `roles` directory at the top level of the repository
  * Requirements need to be specified in particular places
    * Roles should be specified in `roles/requirements.yml`
    * Collections should be specified in `requirements.yml` or `collections/requirements.yml`
    * This appliance uses a single file containing all requirements at
      [requirements.yml](./requirements.yml) which is symlinked from `roles/requirements.yml`
  * If a custom `ansible.cfg` is required, it should be at the top level of the repository

Variables that vary from site-to-site but are fixed for all deployments of the
appliance at a particular site can be set in the `extra_vars` of the `ClusterTemplate` resource. For
example, when deployed in Azimuth, this appliance would require `cluster_image` to be set to
the ID of an Ubuntu 20.04 image on the target cloud.

Azimuth will deal with the mechanics of setting up the required resources using the
`azimuth_caas_cluster_templates_overrides` of the
[azimuth-ops Ansible collection](https://github.com/stackhpc/ansible-collection-azimuth-ops).
For example, to use this appliance in an Azimuth deployment, the following configuration
would be used:

```yaml
azimuth_caas_cluster_templates_overrides:
  sample-appliance:
    # The git URL of the appliance
    gitUrl: https://github.com/stackhpc/azimuth-sample-appliance.git
    # The branch, tag or commit id to use
    gitVersion: master
    # The name of the playbook to use
    playbook: sample-appliance.yml
    # The URL of the metadata file
    uiMetaUrl: https://raw.githubusercontent.com/stackhpc/azimuth-sample-appliance/master/ui-meta/sample-appliance.yml
    # Dict of extra variables for the appliance
    extraVars:
      cluster_image: "<ID of an Ubuntu 20.04 image>"
```

If you are using `infra_community_images` to manage your images as part of the Azimuth deployment,
you can easily use the ID of one of the uploaded images:

```yaml
cluster_image: "{{ infra_community_image_info.ubuntu_2004_20220411 }}"
```

## Ansible variables

When invoking an appliance, Azimuth passes a number of Ansible variables.
These fall into the following groups:

  * **System variables**: Variables derived by Azimuth providing information about the
    environment in which the appliance is being deployed.
  * **User-provided variables**: Variables provided by the user using the form in the
    Azimuth user interface. These are controlled by the cluster metadata file.
  <!-- * **Zenith services**: Variables provided by Azimuth describing the Zenith subdomains
    assigned to the appliance's services. The services are defined in the cluster metadata
    file. These variables are only provided when Zenith is enabled. -->

The following system variables are provided by Azimuth:

| Variable name | Description |
|---|---|
| `cluster_id` | The ID of the cluster. Should be used in the [Terraform state key](./group_vars/openstack.yml#L2). |
| `cluster_name` | The name of the cluster as given by the user. |
| `cluster_type` | The name of the cluster type. |
| `cluster_user_ssh_public_key` | The SSH public key of the user that deployed the cluster. |
| `cluster_deploy_ssh_public_key` | The SSH public key used by AWX to configure the hosts. |
| `cluster_ssh_private_key_file` | The path to a file containing the private key corresponding to `cluster_deploy_ssh_public_key`.<br>This is consumed by the `stackhpc.terraform.infra` role. |
| `cluster_network` | The name of the project internal network onto which cluster nodes should be placed. |
| `cluster_floating_network` | The name of the floating network where floating IPs can be allocated. |
| `cluster_upgrade_system_packages` | This variable is set when a PATCH operation is requested.<br>If given and `true`, it indicates that system packages should be upgraded. If not given, it should be assumed to be `false`.<br>The mechanism for acheiving this is appliance-specific, but it is expected to be a disruptive operation (e.g. rebuilding nodes).<br>If not given or set to `false`, disruptive operations should be avoided where possible. |
| `cluster_state` | This variable is set when a DELETE operation is requested.<br>If given and set to `absent` all cluster resources should be deleted, otherwise cluster resources should be updated as normal. |

## Cluster metadata

Each CaaS appliance has a playbook (which may call other playbooks, roles, etc.) and a
corresponding cluster metadata file. The cluster metadata file provides information about
the cluster such as the human-readable name, logo URL and description. It also defines the
variables that should be collected from the user, including how they should be validated
and rendered in the form that the user sees in the Azimuth UI.

<!-- When Zenith is enabled, the cluster metadata file also specifies the services for which
Zenith subdomains should be allocated and passed to the playbook. -->

The cluster metadata file for this sample appliance is at
[ui-meta/sample-appliance.yml](./ui-meta/sample-appliance.yml). It is heavily documented to
describe the available options.

## Resource provisioning

It is recommended that resources for an appliance are provisioned using Terraform and then adopted into
the in-memory inventory using the
[add_host module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/add_host_module.html).

This appliance uses Terraform to provision security groups, servers and the floating IP and
corresponding association. However it is possible to provision any of the resources supported by
the
[Terraform OpenStack provider](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest/docs).

It is possible to implement this process yourself, however StackHPC provide a role -
[stackhpc.terraform.infra](https://github.com/stackhpc/ansible-collection-terraform/tree/main/roles/infra) -
to simplify the process. This can be used as long as your Terraform outputs conform to a
particular specification - an example of how to use it can be
[seen in this appliance](./roles/cluster_infra/tasks/main.yml). For more details, take a look at the role
documentation and defaults.

## Persistent state

If your appliance requires persistent state, for example a database or
[Ansible local facts](https://docs.ansible.com/ansible/latest/user_guide/playbooks_vars_facts.html#facts-d-or-local-facts),
it is recommended that it is placed onto a volume whose lifecycle is tied to the cluster rather
than any individual machine. This is to ensure that the state is preserved even if the individual
machines are replaced, e.g. during a patch operation.

Creating and attaching the relevant volumes can be done in the Terraform for your appliance.

## Cluster patching

One of the operations available to users in the Azimuth UI is a "patch". The idea of this operation
is that it gives the user control over when packages are updated on their cluster as this is a
potentially disruptive operation.

Patching can be done in two ways:

  1. In-place using the OS package manager (e.g. `yum update -y`)
  2. By replacing the machines with new ones based on an updated image

The second option is preferred, and is implemented by this sample appliance in the
[cluster_infra role](./roles/cluster_infra). This option is preferred because it ensures more
consistency - images can be built and tested in advance before being rolled out to production.
It also allows the use of "fat" images where the majority of the required packages are built
into the image - this can speed up cluster provisioning significantly.

When a patch is requested, the image specified in the `cluster_image` variable is used - this will
force machines to be re-created if the referenced image has been updated. For all other updates,
the image from the previous execution is used. This ensures that all machines in a cluster have
a consistent image, even if they are created at different times (e.g. scaling the number of workers).

## Cluster outputs

When a cluster playbook executes successfully the last task is able to return outputs that can
be presented in the Azimuth UI, in particular using the `usage_template` from the cluster metadata.
To do this, just use a `debug` task with the variable `outputs` set to a dictionary of outputs.

For example, this appliance
[uses the cluster outputs to return the allocated floating IP](./sample-appliance.yml#L29-L34).
