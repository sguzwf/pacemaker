Guest Node Quick Example
------------------------

If you already know how to use Pacemaker, you'll likely be able to grasp this
new concept of guest nodes by reading through this quick example without
having to sort through all the detailed walk-through steps. Here are the key
configuration ingredients that make this possible using libvirt and KVM virtual
guests. These steps strip everything down to the very basics.

.. index::
    single: guest node
    pair: node; guest node

Mile-High View of Configuration Steps
#####################################

* Give each virtual machine that will be used as a guest node a static network
  address and unique hostname.

* Put the same authentication key with the path ``/etc/pacemaker/authkey`` on
  every cluster node and virtual machine. This secures remote communication.

  Run this command if you want to make a somewhat random key:

  .. code-block:: none

     # dd if=/dev/urandom of=/etc/pacemaker/authkey bs=4096 count=1

* Install pacemaker_remote on every virtual machine, enabling it to start at
  boot, and if a local firewall is used, allow the node to accept connections
  on TCP port 3121.

  .. code-block:: none

    # yum install pacemaker-remote resource-agents
    # systemctl enable pacemaker_remote
    # firewall-cmd --add-port 3121/tcp --permanent

  .. NOTE::

      If you just want to see this work, you may want to simply disable the local
      firewall and put SELinux in permissive mode while testing. This creates
      security risks and should not be done on a production machine exposed to the
      Internet, but can be appropriate for a protected test machine.

* Create a Pacemaker resource to launch each virtual machine, using the
  **remote-node** meta-attribute to let Pacemaker know this will be a
  guest node capable of running resources.

  .. code-block:: none

    # pcs resource create vm-guest1 VirtualDomain hypervisor="qemu:///system" config="vm-guest1.xml" meta remote-node="guest1"

  The above command will create CIB XML similar to the following:

  .. code-block:: xml

     <primitive class="ocf" id="vm-guest1" provider="heartbeat" type="VirtualDomain">
       <instance_attributes id="vm-guest-instance_attributes">
         <nvpair id="vm-guest1-instance_attributes-hypervisor" name="hypervisor" value="qemu:///system"/>
         <nvpair id="vm-guest1-instance_attributes-config" name="config" value="guest1.xml"/>
       </instance_attributes>
       <operations>
         <op id="vm-guest1-interval-30s" interval="30s" name="monitor"/>
       </operations>
       <meta_attributes id="vm-guest1-meta_attributes">
         <nvpair id="vm-guest1-meta_attributes-remote-node" name="remote-node" value="guest1"/>
       </meta_attributes>
     </primitive>

In the example above, the meta-attribute **remote-node="guest1"** tells Pacemaker
that this resource is a guest node with the hostname **guest1**. The cluster will
attempt to contact the virtual machine's pacemaker_remote service at the
hostname **guest1** after it launches.

.. NOTE::

    The ID of the resource creating the virtual machine (**vm-guest1** in the above
    example) 'must' be different from the virtual machine's uname (**guest1** in the
    above example). Pacemaker will create an implicit internal resource for the
    pacemaker_remote connection to the guest, named with the value of **remote-node**,
    so that value cannot be used as the name of any other resource.

Using a Guest Node
==================

Guest nodes will show up in ``crm_mon`` output as normal.  For example, this is the
``crm_mon`` output after **guest1** is integrated into the cluster:

.. code-block:: none

    Stack: corosync
    Current DC: node1 (version 1.1.16-12.el7_4.5-94ff4df) - partition with quorum
    Last updated: Fri Jan 12 13:52:39 2018
    Last change: Fri Jan 12 13:25:17 2018 via pacemaker-controld on node1

    2 nodes configured
    2 resources configured

    Online: [ node1 guest1]

    vm-guest1     (ocf::heartbeat:VirtualDomain): Started node1

Now, you could place a resource, such as a webserver, on **guest1**:

.. code-block:: none

    # pcs resource create webserver apache params configfile=/etc/httpd/conf/httpd.conf op monitor interval=30s
    # pcs constraint location webserver prefers guest1

Now, the crm_mon output would show:

.. code-block:: none

    Stack: corosync
    Current DC: node1 (version 1.1.16-12.el7_4.5-94ff4df) - partition with quorum
    Last updated: Fri Jan 12 13:52:39 2018
    Last change: Fri Jan 12 13:25:17 2018 via pacemaker-controld on node1

    2 nodes configured
    2 resources configured

    Online: [ node1 guest1]

    vm-guest1     (ocf::heartbeat:VirtualDomain): Started node1
    webserver     (ocf::heartbeat::apache):       Started guest1

It is worth noting that after **guest1** is integrated into the cluster, nearly all the
Pacemaker command-line tools immediately become available to the guest node.
This means things like ``crm_mon``, ``crm_resource``, and ``crm_attribute`` will work
natively on the guest node, as long as the connection between the guest node
and a cluster node exists. This is particularly important for any promotable
clone resources executing on the guest node that need access to ``crm_master`` to
set transient attributes.
