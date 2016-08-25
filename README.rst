OpenStack-Ansible trove
###############################

Ansible role that installs and configures OpenStack trove. trove is
installed behind the Apache webserver listening on port 9200 by default.

Required Variables
==================

This list is not exhaustive at present. See role internals for further
details.

.. code-block:: yaml

    # trove TCP listening port
    trove_service_port: 9200

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
