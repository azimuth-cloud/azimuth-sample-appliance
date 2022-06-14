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
- [Ansible variables](#ansible-variables)
- [Cluster metadata](#cluster-metadata)
- [Resource provisioning](#resource-provisioning)
- [Cluster outputs](#cluster-outputs)

## Sample appliance architecture

This sample appliance is a simple HTTP service, consisting of one or more backends with
a load-balancer in front.

The load-balancer is an [Nginx](https://nginx.org/) server that is configured to forward
traffic to the backends. The backends are also Nginx servers, and are configured to render
a single page that contains the IP of the host on the internal network. Hence when hitting
the service multiple times, the end user should see a different IP address reported each
time (when more than one backend is configured!).

When [Zenith](https://github.com/stackhpc/zenith) is enabled for the target Azimuth
deployment, the appliance exposes the load-balancer using a Zenith domain. An ephemeral
[SSH jump host](https://wiki.gentoo.org/wiki/SSH_jump_host) with a floating IP is used
to allow Ansible to access the hosts for configuration.

When Zenith is not enabled, the load-balancer is exposed using a floating IP which is
reported to the user using the cluster outputs.

## Ansible variables

When invoking an appliance using an AWX job, Azimuth passes a number of Ansible variables.
These fall into three groups:

  * **System variables**: Variables derived by Azimuth providing information about the
    project into which the appliance is being deployed.
  * **User-provided variables**: Variables provided by the user using the form in the
    user interface. These are controlled by the cluster metadata file.
  * **Zenith services**: Variables provided by Azimuth describing the Zenith subdomains
    assigned to the appliance's services. The services are defined in the cluster metadata
    file. These variables are only provided when Zenith is enabled.

The following system variables are provided by Azimuth:

## Cluster metadata

Each CaaS appliance has a playbook (which may call other playbooks, roles, etc.) and a
corresponding cluster metadata file. The cluster metadata file provides information about
the cluster such as the human-readable name, logo URL and description. It also defines the
variables that should be collected from the user, including how they should be validated
and rendered in the form that the user sees in the Azimuth UI.

When Zenith is enabled, the cluster metadata file also specifies the services for which
Zenith subdomains should be allocated and passed to the playbook.

## Resource provisioning

In this sample appliance, resources are provisioned using Terraform and then adopted into
the in-memory inventory using the
[add_host module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/add_host_module.html).

It is possible to implement this process yourself, however StackHPC provide a role -
[stackhpc.terraform.infra](https://github.com/stackhpc/ansible-collection-terraform/tree/main/roles/infra) -
to simplify the process. This can be used as long as your Terraform outputs conform to a
particular specification - an example of how to use it can be seen in this appliance.
For more details, take a look at the role documentation and defaults.

## Cluster outputs

When a cluster playbook executes successfully, the last task is able to return outputs
that can be consumed by the Azimuth UI.
