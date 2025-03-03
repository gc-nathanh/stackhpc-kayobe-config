================
Cephadm & Kayobe
================

This section describes how to use the Cephadm integration included in StackHPC
Kayobe configuration since Xena to deploy Ceph.

The Cephadm integration takes the form of custom playbooks that wrap
around the Ansible `stackhpc.cephadm collection
<https://galaxy.ansible.com/stackhpc/cephadm>`_ and provide a means to
create or modify Ceph cluster deployments. Supported features are:

-  creating a new cluster from scratch (RedHat/Debian family distros
   supported)
-  creating pools, users, CRUSH rules and EC profiles
-  modifying the OSD spec after initial deployment
-  destroying the cluster

Resources
=========

-  https://docs.ceph.com/en/pacific/cephadm/index.html
-  https://docs.ceph.com/en/pacific/
-  https://docs.ceph.com/en/quincy/cephadm/index.html
-  https://docs.ceph.com/en/quincy/
-  https://github.com/stackhpc/ansible-collection-cephadm

Configuration
=============

Inventory
---------

The collection assumes a set of group entries in Ansible’s inventory.
The following groups are defined in the Kayobe base configuration
inventory ``groups`` file:

-  ``ceph`` (parent for all Ceph nodes)
-  ``mons``
-  ``mgrs``
-  ``osds``
-  ``rgws`` (optional)

Ceph hosts should be added to these groups as appropriate. Typically at
least the ``mons``, ``mgrs``, and ``osds`` groups should be populated.

Example: Separate monitors
~~~~~~~~~~~~~~~~~~~~~~~~~~

For a system with separate monitor hosts, the following could be added
to ``etc/kayobe/environments/<env>/inventory/groups``, to define two
top-level Ceph host groups:

.. code:: ini

   [ceph-mons]
   [ceph-osds]

   [mons:children]
   ceph-mons

   [mgrs:children]
   ceph-mons

   [osds:children]
   ceph-osds

Then, populate the ``ceph-mons`` and ``ceph-osds`` groups with the
necessary hosts, e.g. in
``etc/kayobe/environments/<env>/inventory/hosts``.

Example: Colocated monitors
~~~~~~~~~~~~~~~~~~~~~~~~~~~

For a system with only colocated monitor and OSD hosts, the following
might be appropriate:

NOTE: we are using ``storage`` rather than ``ceph``, since ``ceph``
is already required by the cephadm collection, and redefining it would
introduce a circular dependency between groups.

.. code:: ini

   [storage]

   [mons:children]
   storage

   [mgrs:children]
   storage

   [osds:children]
   storage

Then populate the ``storage`` group with the necessary hosts,
e.g. in ``etc/kayobe/environments/<env>/inventory/hosts``.

Ceph deployment configuration
-----------------------------

Default variables for configuring Ceph are provided in
``etc/kayobe/cephadm.yml``. Many of these defaults will be sufficient,
but you will likely need to set ``cephadm_osd_spec`` to define the OSD
specification.

OSD specification
~~~~~~~~~~~~~~~~~

The following example is a basic OSD spec that adds OSDs for all
available disks:

.. code:: yaml

   cephadm_osd_spec:
     service_type: osd
     service_id: osd_spec_default
     placement:
       host_pattern: "*"
     data_devices:
       all: true

More information about OSD service placement is available
`here <https://docs.ceph.com/en/pacific/cephadm/services/osd/#advanced-osd-service-specifications>`__.

Container image
~~~~~~~~~~~~~~~

The container image to be deployed by Cephadm is defined by
``cephadm_image``, and the tag by ``cephadm_image_tag``. The StackHPC
Kayobe configuration provides defaults for both of these.

Firewalld
~~~~~~~~~

If the Ceph storage hosts are running firewalld, it may be helpful to
set ``cephadm_enable_firewalld`` to ``true`` to enable configuration of
firewall rules for Ceph services.

Ceph post-deployment configuration
----------------------------------

The ``stackhpc.cephadm`` collection also provides roles for
post-deployment configuration of pools, users, CRUSH rules and EC
profiles.

EC profiles
~~~~~~~~~~~

An Erasure Coding (EC) profile is required in order to use Erasure Coded
storage pools. Example EC profile:

.. code:: yaml

   # List of Ceph erasure coding profiles. See stackhpc.cephadm.ec_profiles role
   # for format.
   cephadm_ec_profiles:
     - name: ec_4_2_hdd
       k: 4
       m: 2
       crush_device_class: hdd

CRUSH rules
~~~~~~~~~~~

CRUSH rules may not be required in a simple setup with a homogeneous
pool of storage. They are useful when there are different tiers of
storage. The following example CRUSH rules define separate tiers for
Hard Disk Drives (HDDs) and Solid State Drives (SSDs).

.. code:: yaml

   # List of Ceph CRUSH rules. See stackhpc.cephadm.crush_rules role for format.
   cephadm_crush_rules:
     - name: replicated_hdd
       bucket_root: default
       bucket_type: host
       device_class: hdd
       rule_type: replicated
       state: present
     - name: replicated_ssd
       bucket_root: default
       bucket_type: host
       device_class: ssd
       rule_type: replicated
       state: present

Pools
~~~~~

The following example pools should be sufficient to work with the
default `external Ceph
configuration <https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html>`__
for Cinder, Cinder backup, Glance, and Nova in Kolla Ansible.

.. code:: yaml

   # List of Ceph pools. See stackhpc.cephadm.pools role for format.
   cephadm_pools:
     - name: backups
       application: rbd
       state: present
     - name: images
       application: rbd
       state: present
     - name: volumes
       application: rbd
       state: present
     - name: vms
       application: rbd
       state: present

If a pool needs to use a particular CRUSH rule, this can be defined via
``rule_name: <rule>``.

Keys
~~~~

The following example keys should be sufficient to work with the default
`external Ceph
configuration <https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html>`__
for Cinder, Cinder backup, Glance, and Nova in Kolla Ansible.

.. code:: yaml

   # List of Cephx keys. See stackhpc.cephadm.keys role for format.
   cephadm_keys:
     - name: client.cinder
       caps:
         mon: "profile rbd"
         osd: "profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images"
         mgr: "profile rbd pool=volumes, profile rbd pool=vms"
     - name: client.cinder-backup
       caps:
         mon: "profile rbd"
         osd: "profile rbd pool=volumes, profile rbd pool=backups"
         mgr: "profile rbd pool=volumes, profile rbd pool=backups"
     - name: client.glance
       caps:
         mon: "profile rbd"
         osd: "profile rbd pool=images"
         mgr: "profile rbd pool=images"
       state: present

Ceph Commands
~~~~~~~~~~~~~

It is possible to run an arbitrary list of commands against the cluster after
deployment by setting the ``cephadm_commands_pre`` and ``cephadm_commands_post``
variables. Each should be a list of commands to pass to ``cephadm shell --
ceph``. For example:

.. code:: yaml

   # A list of commands to pass to cephadm shell -- ceph. See stackhpc.cephadm.commands
   # for format.
   cephadm_commands_pre:
    # Configure Prometheus exporter to listen on a specific interface. The default
    # is to listen on all interfaces.
    - "config set mgr mgr/prometheus/server_addr 10.0.0.1"

Both variables have the same format, however commands in the
``cephadm_commands_pre`` list are executed before the rest of the Ceph
post-deployment configuration is applied. Commands in the
``cephadm_commands_post`` list are executed after the rest of the Ceph
post-deployment configuration is applied.

Manila & CephFS
~~~~~~~~~~~~~~~

Using Manila with the CephFS backend requires the configuration of additional
resources.

A Manila key should be added to cephadm_keys:

.. code:: yaml

  # Append the following to cephadm_keys:
  - name: client.manila
    caps:
      mon: "allow r"
      mgr: "allow rw"
    state: present

A CephFS filesystem requires two pools, one for metadata and one for data:

.. code:: yaml

  # Append the following to cephadm_pools:
  - name: cephfs_data
    application: cephfs
    state: present
  - name: cephfs_metadata
    application: cephfs
    state: present

Finally, the CephFS filesystem itself should be created:

.. code:: yaml

  # Append the following to cephadm_commands_post:
  - "fs new manila-cephfs cephfs_metadata cephfs_data"
  - "orch apply mds manila-cephfs"

In this example, the filesystem is named ``manila-cephfs``. This name
should be used in the Kolla Manila configuration e.g.:

.. code:: yaml

  manila_cephfs_filesystem_name: manila-cephfs

RADOS Gateways
--------------

RADOS Gateways (RGWs) are defined with the following:

.. code:: yaml

  cephadm_radosgw_services:
    - id: myrgw
      count_per_host: 1
      spec:
        rgw_frontend_port: 8100

The port chosen must not conflict with any other processes running on the Ceph
hosts. Port 8100 does not conflict with our default suite of services.

Ceph RGWs require additional configuration to:

  * Support both S3 and Swift APIs.

  * Authenticate user access via Keystone.

  * Allow cross-project and public object access.

The set of commands below configure all of these.

.. code:: yaml

  # Append the following to cephadm_commands_post:
  - "config set client.rgw rgw_content_length_compat true"
  - "config set client.rgw rgw_enable_apis 's3, swift, swift_auth, admin'"
  - "config set client.rgw rgw_enforce_swift_acls true"
  - "config set client.rgw rgw_keystone_accepted_admin_roles 'admin'"
  - "config set client.rgw rgw_keystone_accepted_roles 'member, Member, _member_, admin'"
  - "config set client.rgw rgw_keystone_admin_domain Default"
  - "config set client.rgw rgw_keystone_admin_password {{ secrets_ceph_rgw_keystone_password }}"
  - "config set client.rgw rgw_keystone_admin_project service"
  - "config set client.rgw rgw_keystone_admin_user 'ceph_rgw'"
  - "config set client.rgw rgw_keystone_api_version '3'"
  - "config set client.rgw rgw_keystone_token_cache_size '10000'"
  - "config set client.rgw rgw_keystone_url https://{{ kolla_internal_fqdn }}:5000"
  - "config set client.rgw rgw_keystone_verify_ssl false"
  - "config set client.rgw rgw_max_attr_name_len '1000'"
  - "config set client.rgw rgw_max_attr_size '1000'"
  - "config set client.rgw rgw_max_attrs_num_in_req '1000'"
  - "config set client.rgw rgw_s3_auth_use_keystone true"
  - "config set client.rgw rgw_swift_account_in_url true"
  - "config set client.rgw rgw_swift_versioning_enabled true"

As we have configured Ceph to respond to Swift APIs, you will need to tell
Kolla to account for this when registering Swift endpoints with Keystone. Also,
when ``rgw_swift_account_in_url`` is set, the equivalent Kolla variable should
be set in Kolla ``globals.yml`` too:

.. code:: yaml

  ceph_rgw_swift_compatibility: false
  ceph_rgw_swift_account_in_url: true

``secrets_ceph_rgw_keystone_password`` should be stored in the Kayobe
``secrets.yml``, and set to the same value as ``ceph_rgw_keystone_password`` in
the Kolla ``passwords.yml``. As such, you will need to configure Keystone
before deploying the RADOS gateways. If you are using the Kolla load balancer
(see :ref:`RGWs-with-hyper-converged-Ceph` for more info), you can specify the
``haproxy`` and ``loadbalancer`` tags here too.

.. code:: yaml

  kayobe overcloud service deploy -kt ceph-rgw,keystone,haproxy,loadbalancer


.. _RGWs-with-hyper-converged-Ceph:

RGWs with hyper-converged Ceph
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you are using a hyper-converged Ceph setup (i.e. your OpenStack controllers
and Ceph storage nodes share the same hosts), you should double-check that
``rgw_frontend_port`` does not conflict with any processes on the controllers.
For example, port 80 (and 443) will be bound to the Kolla-deployed haproxy. You
should choose a custom port that does not conflict with any OpenStack endpoints
too (``openstack endpoint list``).

You may also want to use the Kolla-deployed haproxy to load balance your RGWs.
This means you will not need to define any Ceph ingress services. Instead, you
add definitions of your Ceph hosts to Kolla ``globals.yml``:

.. code:: yaml

  ceph_rgw_hosts:
    - host: controller1
      ip: <host IP on storage net>
      port: 8100
    - host: controller2
      ip: <host IP on storage net>
      port: 8100
    - host: controller3
      ip: <host IP on storage net>
      port: 8100

HA with Ingress services
~~~~~~~~~~~~~~~~~~~~~~~~

Ingress services are defined with the following. ``id`` should match the name
(not id) of the RGW service to which ingress will point to. ``spec`` is a
service specification required by Cephadm to deploy the ingress (haproxy +
keepalived pair).

Note that the ``virtual_ip`` here must be different than the Kolla VIP. The
choice of subnet will be dependent on your deployment, and can be outside
of any Ceph networks.

.. code:: yaml

  cephadm_ingress_services:
    - id: rgw.myrgw
      spec:
        frontend_port: 443
        monitor_port: 1967
        virtual_ip: 10.66.0.1/24
        ssl_cert: {example_certificate_chain}

When using ingress services, you will need to stop Kolla from configuring your
RGWs to use the Kolla-deployed haproxy. Set the following in Kolla
``globals.yml``:

.. code:: yaml

  enable_ceph_rgw_loadbalancer: false

Deployment
==========

Host configuration
------------------

Configure the Ceph hosts:

.. code:: bash

   kayobe overcloud host configure --limit storage

Ceph deployment
---------------

..
  **FIXME**: Wait for Ceph to come up, so that we can just run cephadm.yml

Deploy the Ceph services:

.. code:: bash

   kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/cephadm-deploy.yml

You can check the status of Ceph via Cephadm on the storage nodes:

.. code:: bash

   sudo cephadm shell -- ceph -s

Once the Ceph cluster has finished initialising, run the full
cephadm.yml playbook to perform post-deployment configuration:

.. code:: bash

   kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/cephadm.yml

The ``cephadm.yml`` playbook imports various other playbooks, which may also be
run individually to perform specific tasks. Note that if you want to deploy
additional services (such as RGWs or ingress) after an initial deployment, you
will need to set ``cephadm_bootstrap`` to true. For example:

.. code:: bash

   kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/cephadm.yml -e cephadm_bootstrap=true

Configuration generation
------------------------

Generate keys and configuration for Kolla Ansible:

.. code:: bash

   kayobe playbook run $KAYOBE_CONFIG_PATH/ansible/cephadm-gather-keys.yml

This will generate Ceph keys and configuration under
``etc/kayobe/environments/<env>/kolla/config/``, which should be
committed to the configuration.

This configuration will be used during
``kayobe overcloud service deploy``.
