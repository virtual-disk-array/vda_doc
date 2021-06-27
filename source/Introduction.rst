Introduction
============

Overview
--------

`VDA <https://github.com/virtual-disk-array/vda>`_ (Virtual Disk Array)
is a distributed storage system. It uses `SPDK <https://spdk.io/>`_ as
data plane. It could create and manage a set of virtual disk
arrays. The virtual disk array looks like a traditional disk array,
which has one or multiple controllers and several disks. The user can
set raid level, encryption or other features to the disk array
(currently, only raid0 is supported). In VDA, all the controllers and
disks are SPDK applications. They are connected by NVMeOF. A typical
usage of VDA is as persistent volumes for kubernets.

.. image:: /images/vda_cluster_arch.png

Components
----------

.. _portal-label:

portal
^^^^^^
The portal is a set of API servers. It accepts gRPC calls. The client
use it to create/delete/manage the virtual disk arrays and other related
resources.

.. _etcd-label:

etcd
^^^^
The `etcd <https://etcd.io/>`_ cluster is used to stores all the data
and statements. E.g., it stores the configuration of the virtual disk
arrays, the disk node / controller node status. It is also used as a
coordinator. Multiple monitors register themselves to the etcd and
then they can arrange their tasks.

.. _monitor-label:

monitor
^^^^^^^
It syncs up data from the etcd cluster to all the disk nodes (DN) and
controller nodes (CN), gets current status from disk nodes and
controller nodes, and updates their status to the etcd. It also detects
the failure of cnotroller nodes, then performs failover.

.. _cn-label:

Controller Node (CN)
^^^^^^^^^^^^^^^^^^^^
A disk array (DA) has one or more controllers. When a user creates a
DA, the portal allocates controllers from CNs. Each CN should run two
softwares: SPDK applicaton and cn_agent.

.. _dn-label:

Disk Node (DN)
^^^^^^^^^^^^^^
A disk array (DA) has one or more disks. When au ser creates a DA, the
portal allocates disks from DNs. Each DN should run two softwares:
SPDK application and dn_agent.

.. _cn-agent-label:

cn_agent
^^^^^^^^
The cn_agent runs on the Controller Node (CN). It accepts rpc from
portal and monitor, sends API to spdk application, creates/deletes
resources about the controllers of the disk arrays (DA).

.. _dn-agent-label:

dn_agent
^^^^^^^^
The dn_agent runs on the Disk Node (DN). It accepts rpc from portal
and monitor, sends API to spdk application, creates/deletes resources
about the disks of the disk array.

View from the data plane
------------------------

Below diagram shows the components of a single disk array (DA).

.. image:: /images/vda_disk_array_arch.png


From the host (the user of the DA) perspective, the DA is a multiple
controllers disk array, and both of the two controllers are
active. The host connects to the controllers via NVMeOF and the host
could write data to each of the controllers. But from the intiernal
perspective, there is a primary controller, only the primary
controller can write data to the disk(s). If the host writes to the
secondary controller, the secondary controller will send the data to
the priarmy, and let the priamry write data to the disk.  The
controllers are allocated from the Controller Node (CN). The primary
controller aggregates multiple virtual disks (VD) to a single disk,
creates a logical volume store (lvs) on it and create a logical volume
from the lvs. The logical volume can be export to a host via a
exporter (EXP). The connection between the host and the exp is
NVMeOF. Each VD is allocated from a physical disk (PD), the PD is
managed by the data node (DN).  The connection between VD and
controller is NVMeOF too.

.. _pd-label:

Physical Disk (PD)
^^^^^^^^^^^^^^^^^^
A PD is a bdev in the SPDK application. Generally, it indicates a
physical disk controlled by the disk node (DN). A `SPDK logical volume store <https://spdk.io/doc/logical_volumes.html#lvs>`_
is created on top of the bdev. When a disk array (DA) allocates a disk
from the PD, a logical volume will be created from the logical volume
store (which we call it VD, please refer the later part of this doc),
and the logical volume will be provided to the DA. Currently VDA
supports 3 kind of PD:

#. `nvme <https://spdk.io/doc/bdev.html#bdev_config_nvme>`_
#. `aio <https://spdk.io/doc/bdev.html#bdev_config_aio>`_
#. `malloc <https://spdk.io/doc/bdev.html#bdev_config_malloc>`_

.. _vd-label:

Virtual Disk (VD)
^^^^^^^^^^^^^^^^^
A VD is a `logical volume <https://spdk.io/doc/logical_volumes.html#lvol>`_
allocated from the PD. It is exported to the controller node (CN) as an
NVMeOF target. The VD stores user data of a DA.

.. _cntlr-label:

Controller (cntlr)
^^^^^^^^^^^^^^^^^^
The cntlr has two kind of role: primary and secondary. A DA can only
have a single primary cntlr and 0 or more secondary cntlrs. The
primary cntlr connects to all the VDs, aggregate them to a single
block device (a bdev in SPDK), then create a `logical volume store <https://spdk.io/doc/logical_volumes.html#lvs>`_,
and create a default logical volume from the logical volume store. The
default logical volume can be exported to the host. In the future, VDA
will support snapshot, then a user can create snapshot from the
default logical volume and export the snapshot too. The primary cntlr
export the default logicla volume to both the host and the secondary
cntlr(s). The secondary cntlr connect to the primary cntlr, and export
the NVMeOF device to the host too. So the host doesn't care which one
is primary and which one is secondary. If a CN doesn't work, the
primary cntlrs on this CN will be changed to secondary, and the
secondary cntlrs on other CNs will be promote to primary.

.. _exp-label:

Exporter (EXP)
^^^^^^^^^^^^^^
The EXP is a NVME subsystem. It has a single NVME namespace which
represent the DA. When a user want to let a host connect to the DA,
the user should create an EXP.

.. _da-label:

Disk Array (DA)
^^^^^^^^^^^^^^^
The DA is a virtual disk array. It has multiple controllers and
could be accessed by the NVMeOF protocal.

.. _host-label:

Host
^^^^
The user or client of the VDA. It is a NVMeOF initiator.
