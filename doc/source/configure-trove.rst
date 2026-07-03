=================
Configuring Trove
=================

.. note::

   Care should be taken when deploying Trove in production environments.
   Be sure to fully understand the security implications of the deployed
   architecture.

Trove provides DBaaS to an OpenStack deployment. It deploys guest instances
that provide the desired DB for use by the end consumer. The trove
guest instances need connectivity back to the trove services via RPC
(oslo.messaging) and the OpenStack services. The way these guest instances
get access to those services could be via internal networking (in the
case of oslo.messaging) or via public interfaces (in the case of
OpenStack services). For the example configuration, we'll designate a
provider network as the network for Trove to provision on each guest
instance. The guest can then connect to oslo.messaging via this network and
to the OpenStack services externally. Optionally, the guest instances could
use the internal network to access OpenStack services, but that would require
more containers being bound to this network.

The deployment configuration outlined below may not be appropriate for
production environments. Review this very carefully with your own security
requirements.

Setup a Neutron network for use by Trove
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Trove needs connectivity between the control plane and the DB guest instances.
For this purpose a provider network should be created which bridges the Trove
containers (if the control plane is installed in a container) or hosts with
instances. In a general case, neutron networking can be a simple flat network.
An example entry into ``openstack_user_config.yml`` is shown below:

.. code-block:: yaml

     # OVS deployment
     - network:
        container_bridge: "br-dbaas"
        container_type: "veth"
        container_interface: "eth13"
        host_bind_override: "eth13"
        ip_from_q: "dbaas"
        type: "flat"
        net_name: "dbaas-mgmt"
        group_binds:
          - neutron_openvswitch_agent
          - rabbitmq
      # OVN deployment
      - network:
        container_bridge: "br-dbaas"
        container_type: "veth"
        container_interface: "eth13"
        host_bind_override: "eth13"
        ip_from_q: "dbaas"
        type: "flat"
        net_name: "dbaas-mgmt"
        group_binds:
          - neutron_ovn_gateway
          - rabbitmq

Make sure to modify the other entries in this file as well.

The ``net_name`` will be the physical network that is specified when creating
the neutron network. The default value of ``dbaas-mgmt`` is also used to
lookup the addresses of the rpc messaging container. If the default is not used
then some variables in ``defaults\main.yml`` will need to be overwritten.

By default this role will not create the neutron network automaticaly. However,
the default values can be changed to create the neutron network. If you want the
role to configure the network during deployment, set trove_service_net_setup to
true and adjust the other trove_service_net_* variables as needed. See the
``trove_service_net_*`` variable in ``defaults\main.yml``:

.. code-block: yaml

   trove_service_net_setup: false
   trove_service_net_validate_certs: false
   trove_service_net_phys_net: dbaas-mgmt
   trove_service_net_type: flat
   trove_service_net_name: dbaas_service_net
   # Network segmentation ID if vlan, gre...
   trove_service_net_segmentation_id:
   trove_service_subnet_name: dbaas_subnet
   trove_service_net_subnet_cidr: "172.29.252.0/22"
   trove_service_net_dhcp: true
   trove_service_net_allocation_pool_start: "172.29.252.110"
   trove_service_net_allocation_pool_end: "172.29.255.254"

By customizing the ``trove_service_net_*`` variables and having this role
create the neutron network a full deployment of the OpenStack and DBaaS can
proceed without interruption or intervention.

The following is an example how to set up a provider network in neutron
manually, if so desired:

.. code-block:: bash

    neutron net-create dbaas_service_net --shared \
                                    --provider:network_type flat \
                                    --provider:physical_network dbaas-mgmt

    neutron subnet-create dbaas_service_net 172.29.252.0/22 --name dbaas_service_subnet
                          --ip-version=4 \
                          --allocation-pool start=172.29.252.110,end=172.29.255.255 \
                          --enable-dhcp \
                          --dns-nameservers list=true 8.8.4.4 8.8.8.8

Special attention needs to be applied to the ``--allocation-pool`` to not have
ips which overlap with ips assigned to hosts or containers (see the ``used_ips``
variable in ``openstack_user_config.yml``)

.. note::
    This role needs the neutron network created before it can run properly
    since the trove guest agent configuration file contains that information.


Building Trove images
~~~~~~~~~~~~~~~~~~~~~

When building disk image for the guest instances deployments there are many
items to consider. Listed below are a few:

#. Security of the instance and the network infrastructure
#. What DBs will be installed
#. What DB services will be supported
#. How will the images be maintained

Images can be built using the ``diskimage-builder`` tooling. The trove
virtual environment can be tar'd up from the trove containers and deployed to
the images using custom ``diskimage-builder`` elements.

See the ``trove/integration/scripts/files/elements`` directory contents in
the OpenStack Trove project for ``diskimage-builder`` elements to build trove
disk images.


Use stand-alone RabbitMQ
~~~~~~~~~~~~~~~~~~~~~~~~

Since Trove uses RabbitMQ to interact with guest servers it requires you to
pass the neutron network into the RabbitMQ container which is a security risk.
As a result, you might want to isolate Trove from other services in terms of
the RabbitMQ cluster and use a standalone one.

Architecture overview
---------------------

The standalone cluster exposes two separate paths to the same broker:

* **Control plane** (Trove API / conductor containers) -> standalone RabbitMQ
  via the **management network** (i.e. br-mgmt).
* **Guest agents** (database instances) -> standalone RabbitMQ via the **dbaas
  provider network** (``trove_provider_network``).

This means Trove service containers don't need dbaas interface — they reach
the broker over management. Only the standalone RabbitMQ containers require a
dbaas interface to have connection with guest instances.

Depending on your security requirements, you may instead want the standalone
RabbitMQ to have no management interface at all and serve both the control
plane and the guest agents over the dbaas network only. This avoids
the instance facing dbaas network into the sensitive management
network through the RabbitMQ, at the cost of additional configuration (the
Trove service containers must then reach the broker over dbaas, and the
cluster must be deployable without a management address). Which layout is
preferable depends on your isolation goals.

.. note::

   The guest **notification** transport is also routed through the standalone
   cluster. If Ceilometer is used, its central agent must likewise be connected
   to this network and provided with the appropriate overrides to consume
   notification messages from the standalone cluster - the deployer is responsible
   for configuring this.

In order to deploy new RabbitMQ cluster and use it for Trove, you will need
to:

#. Create a new group for RabbitMQ containers. You will need to create a file
   inside ``/etc/openstack_depoy/env.d`` which defines group mappings:

    .. code-block:: yaml

        component_skel:
          trove_rabbitmq:
            belongs_to:
              - trove_mq_all

        container_skel:
          trove_rabbit_container:
            belongs_to:
              - trove_mq_containers
            contains:
              - trove_rabbitmq

        physical_skel:
          trove_mq_containers:
            belongs_to:
              - all_containers
          trove_mq_hosts:
            belongs_to:
              - hosts

#. Define on which hosts this group will be deployed. This can be done either
   with a new file in ``conf.d`` or inside ``openstack_user_config.yml``:

    .. code-block:: yaml

        trove_mq_hosts:
          aio1:
            ip: 172.29.236.100

#. Add to the dbaas network mapping for the new group.
   Add a **single** ``provider_networks`` entry binding the dbaas network to
   ``trove_rabbitmq`` only. The control plane reaches the broker over the
   management network, so only the standalone RabbitMQ containers need a dbaas
   interface (for guest instances to reach them).

    .. code-block:: yaml

        - network:
            container_bridge: "br-dbaas"
            container_type: "veth"
            container_interface: "eth14"
            ip_from_q: "dbaas"
            type: "raw"
            net_name: "dbaas-mgmt"
            group_binds:
              - trove_rabbitmq

#. Create override for dedicated RabbitMQ containers, ie
   ``/etc/openstack_deploy/group_vars/trove_rabbitmq.yml``

    .. code-block:: yaml

        rabbitmq_cluster_name: trove

#. Create overrides for trove service containers, ie
   ``/etc/openstack_deploy/group_vars/trove_all.yml``

    .. note::

        For notifications we still want to use main RabbitMQ cluster

    .. code-block:: yaml

       # Point the control-plane RPC vhost/user setup at the standalone cluster.

       trove_oslomsg_rpc_host_group: trove_rabbitmq

       # Point guest agent RPC at the standalone
       # cluster via the dbaas provider network IPs.
       trove_guest_rpc_host_group: trove_rabbitmq

#. Add required overrides to user_variables.yml:

    .. code-block:: yaml

        # trove_oslomsg_rpc_servers defaults to the global
        # oslomsg_rpc_servers (main RabbitMQ) and is not derived from
        # trove_oslomsg_rpc_host_group.  Without this override, trove.conf
        # transport_url points at the main cluster even though guestagent.conf
        # correctly targets the standalone one.
        trove_oslomsg_rpc_servers: >-
          {{ groups['trove_rabbitmq'] | map('extract', hostvars)
             | map(attribute='ansible_host') | join(',') }}

        # Align the control-plane RabbitMQ password with the one written
        # to guestagent.conf.
        trove_oslomsg_rpc_password: "{{ trove_guest_oslomsg_rpc_password }}"

#. Generate the standalone RabbitMQ password

``trove_guest_oslomsg_rpc_password`` must exist as a concrete value in
``user_secrets.yml`` before running any playbook.
Add the key and generate its value:

    .. code-block:: bash

       echo "trove_guest_oslomsg_rpc_password:" >> /etc/openstack_deploy/user_secrets.yml
       /opt/openstack-ansible/scripts/pw-token-gen.py --file /etc/openstack_deploy/user_secrets.yml

#. Run playbooks to create RabbitMQ containers and deploy cluster on them

    .. code-block:: bash

        openstack-ansible openstack.osa.containers_lxc_create --limit trove_rabbitmq,lxc_hosts
        openstack-ansible openstack.osa.rabbitmq_server -e rabbitmq_host_group=trove_rabbitmq
