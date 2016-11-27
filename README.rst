========================
Team and repository tags
========================

.. image:: http://governance.openstack.org/badges/openstack-ansible-os_trove.svg
    :target: http://governance.openstack.org/reference/tags/index.html

.. Change things from this point on

OpenStack-Ansible Trove (DBaaS)
###############################

Ansible role that installs and configures OpenStack Trove. Trove is
installed behind the Apache webserver listening on port 8779 by default.

Required Variables
==================

This list is not exhaustive at present. See role internals for further
details.

.. code-block:: yaml

    # trove TCP listening port
    trove_service_port: 8779

Example Playbook
================

.. code-block:: yaml

   - name: Install trove server
     hosts: trove_all
     user: root
     roles:
        - { role: "os_trove", tags: [ "os-trove" ] }
     vars:
       is_metal: "{{ properties.is_metal|default(false) }}"
