=======================
OpenStack-Ansible Trove
=======================

This Ansible role installs Trove.

To clone or view the source code for this repository, visit the role repository
for `os_trove <https://github.com/openstack/openstack-ansible-os_trove>`_.

.. toctree::
   :maxdepth: 2

   configure-trove.rst

Default variables
~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../defaults/main.yml
  :language: yaml
  :start-after: under the License.

Required variables
~~~~~~~~~~~~~~~~~~

This list is not exhaustive at present. See role internals for further
details.

.. code-block:: yaml

    # Service and user passwords
    trove_galera_password:
    trove_rabbitmq_password:
    trove_service_password:
    trove_admin_user_password:
    trove_regular_user_password:

    # Trove RPC encryption keys.
    trove_taskmanager_rpc_encr_key:
    trove_inst_rpc_key_encr_key:


Dependencies
~~~~~~~~~~~~

None.

Example playbook
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../examples/playbook.yml
  :language: yaml
