..
    Convention for heading levels:
    =======  Heading 0 (reserved for the title in a document)
    -------  Heading 1
    ~~~~~~~  Heading 2
    +++++++  Heading 3
    '''''''  Heading 4
    (Avoid deeper levels because they do not render well.)


Deployment Tool Integration
==============================

The networking-ovn git repository includes integration with DevStack, which
enables the creation of simple development and test environments with OVN.  The
use of OVN in a realstic deployment requires integration with OpenStack
deployment tooling.

The purpose of this guide is to document what’s required to integrate OVN into
an OpenStack deployment tool.  It discusses OpenStack nodes of 3 different
types:

* **Controller Node** - A node that runs OpenStack control services such as REST
  APIs and databases.

* **Network Node** - A node that runs the Neutron L3 agent and provides routing
  between tenant networks, as well as connectivity to an external network.
  This node may also be running the Neutron DHCP agent to provide DHCP services
  to tenant networks.

* **Compute Node** - A hypervisor.

New Packages
---------------

The Neutron integration for OVN is an independent package, ``networking-ovn``.

OVN is a part of OVS.  The first release that includes OVN is OVS 2.5, though
OVN is technically experimental in that release.  OVN gets installed
automatically if you install OVS from source.  The OVS RPM includes OVN as a
sub-package called ``openvswitch-ovn``.  The Debian/Ubuntu packaging has not
been updated for OVN yet.

Controller Nodes
-------------------

Controller nodes should have both the ``networking-ovn`` and ``openvswitch-ovn``
packages installed.

ovn-northd and ovsdb-server
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

OVN has two databases both managed by ``ovsdb-server``.  It also has a control
service, ``ovn-northd``.

To start both ``ovsdb-server`` and ``ovn-northd``, you can use the ovn-northd
systemd unit::

    $ sudo systemctl start ovn-northd

Or you can start it using the ``ovn-ctl`` script::

    $ sudo /usr/share/openvswitch/scripts/ovn-ctl start_northd

There should only be a single instance of ``ovn-northd`` and ``ovsdb-server``
running. For HA, you can run them in an active/passive mode.  See the HA section
of the networking-ovn FAQ for more information about the current state and
future plans around HA for the control services.

``ovsdb-server`` must be told to listen via TCP so that compute nodes will be
able to connect to the database.  You can enable that with the following
command::

    $ sudo ovs-appctl -t ovsdb-server ovsdb-server/add-remote ptcp:6640:IP_ADDRESS

``IP_ADDRESS`` should be the address that remote services use to connect to
``ovsdb-server`` running on this node.  TCP port 6640 must be made accessible if a
firewall is in use.

neutron-server
~~~~~~~~~~~~~~~~~

OVN has its own Neutron core plugin.  ``neutron-server`` must be configured to
use this new plugin.  The following settings should be applied to
``/etc/neutron/neutron.conf``::

    [DEFAULT]
    core_plugin = networking_ovn.plugin.OVNPlugin
    service_plugins =
         iniset $Q_PLUGIN_CONF_FILE ovn ovsdb_connection “$OVN_REMOTE”
         iniset $Q_PLUGIN_CONF_FILE ovn ovn_l3_mode “$OVN_L3_MODE”

The following options should be set in
``/etc/neutron/plugins/networking-ovn/networking-ovn.ini``::

    [ovn]
    ovsdb_connection = tcp:IP_ADDRESS:6640
    # If running in an active/passive HA mode, you’ll want to add this setting:
    # neutron_sync_mode = repair

``IP_ADDRESS`` should match the one used when configuring ``ovsdb-server``
earlier.

Network Nodes
----------------
