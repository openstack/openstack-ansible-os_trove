=======================
OpenStack-Ansible Trove
=======================

This Ansible role installs Trove.

To clone or view the source code for this repository, visit the role repository
for `os_trove <https://opendev.org/openstack/openstack-ansible-os_trove>`_.

.. toctree::
   :maxdepth: 2

   configure-trove.rst

Default variables
~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../defaults/main.yml
  :language: yaml
  :start-after: under the License.

Dependencies
~~~~~~~~~~~~

To use this role, define the following variables:

.. code-block:: yaml

    # trove TCP listening port
    trove_service_port: 8779

    # Service and user passwords
    trove_galera_password:
    trove_rabbitmq_password:
    trove_service_password:
    trove_admin_user_password:

    # Trove RPC encryption keys.
    trove_taskmanager_rpc_encr_key:
    trove_inst_rpc_key_encr_key:
    trove_instance_rpc_encr_key:

This list is not exhaustive at present. See role internals for further
details.

Example playbook
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../examples/playbook.yml
  :language: yaml
