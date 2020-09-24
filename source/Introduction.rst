Introduction
============

Overview
--------

`VDA <https://github.com/virtual-disk-array/vda>`_ (Virtual Disk Array)
is a distributed storage system. It uses `SPDK <https://spdk.io/>`_ as
data plane. It could create and manage a set of virtual disk
arrays. The virtual disk array looks like a traditional disk array,
which has one or multiple controllers and several disks. The user can
set raid level, encryption or other features on the controller
(currently, only raid0 is supported). In VDA, all the controllers and
disks are SPDK applications. They are connected by NVMeOF. The virtual
disk array could be used as persistent volumes of kubernets or
openstack cinder backend driver. Below is the architecture of VDA:

.. image:: /images/vda_cluster_arch.png

Components
----------

portal
^^^^^^
The portal is a set of API servers. It is the user interface. It
accepts gRPC. The client use it to create/delete/manage the virtual
disk arrays and other resources.

database
^^^^^^^^
The central storage for all the cluster data. It could be MySQL,
Postgresql or sqlite.

monitor
^^^^^^^
It syncs up data from database to all the disk node (dn) and controller
node (cn), gets current status from disk node and controller node,
and updates the status to the database. It also detects the failure of
cnotroller node, and performs failover.

Controller Node (cn)
^^^^^^^^^^^^^^^^^^^^
The VDA allocates controllers from cn. One cn would have multiple
controllers for different virtual disk arrays. Every CN should run two
softwares: spdk application and cn_agent.

Disk Node (dn)
^^^^^^^^^^^^^^
The VDA allocates disks from dn. One dn would provide disks for
different virtual disk arrays. Every dn should run two softwares: spdk
application and dn_agent.

cn_agent
^^^^^^^^
The cn_agent runs on the Controller Node (cn). It accepts rpc from
portal and monitor, sends API to spdk application, creates/deletes
resources for the controllers of the virtual disk arrays.

dn_agent
^^^^^^^^
The dn_agent runs on the Disk Node (dn). It accepts rpc from portal
and monitor, sends API to spdk application, creates/deletes resources
for the disks of the virtual disk array.

View from the data plane
------------------------

Below diagram shows the components of a single virtual disk array.

.. image:: /images/vda_disk_array_arch.png


Physical Disk (pd)
^^^^^^^^^^^^^^^^^^
A pd is a bdev in the SPDK application. It indicates a physical disk
provided by the dn. Each pd is also a `logical volume store <https://spdk.io/doc/logical_volumes.html#lvs>`_
in spdk. The VDA cluster allocate disks for the virtual disk arrays
from the logical volume store. Currently VDA supports 3 kind of pd,
they are

#. `nvme <https://spdk.io/doc/bdev.html#bdev_config_nvme>`_
#. `aio <https://spdk.io/doc/bdev.html#bdev_config_aio>`_
#. `malloc <https://spdk.io/doc/bdev.html#bdev_config_malloc>`_

Virtual Disk (vd)
^^^^^^^^^^^^^^^^^
A vd is a `logical volume <https://spdk.io/doc/logical_volumes.html#lvol>`_
allocated from pd. And it is exported as an NVMeOF target. It stores
the data of the virtual disk arrays.

Controller (cntlr)
^^^^^^^^^^^^^^^^^^
The cntlr has two kind of role: primary and secondary. A virtual disk
array can only have a single primary cntlr and 0 or more secondary
cntlrs. The primary cntlr connects to all the vds, aggregate them to a
single block device (a bdev in SPDK). And then it exports it to the
host and other secondary cntlr(s).
The secondary cntlr connects to the primary cntlr, and then exports it
to the host. So from the host prespective, both primary and secondary
are accessable. The multiple cntlrs of the virtual disk array are
active/active mode. If a cn doesn't work, a monitor would find it, and
the monitor  will perform ailover for all the cntlrs in this cn. In
the above diagram, cntlr0 is the primary cntlr, cntlr1 is the
secondary cntlr.

Disk Array (da)
^^^^^^^^^^^^^^^
The da is a virtual disk array. It has multiple controllers and
could be accessed by the NVMeOF protocal.

Host
^^^^
The user or client of the VDA. It is a NVMeOF initiator.
