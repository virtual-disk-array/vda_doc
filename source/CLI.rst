CLI
===
The vda_cli exports all the
`portal grpc <https://github.com/virtual-disk-array/vda/blob/master/pkg/proto/portalapi/portalapi.proto>`_.

vda_cli dn
----------
The ``vda_cli dn`` sub-command is used to manage the :ref:`DN <dn-label>`.

vda_cli dn create
^^^^^^^^^^^^^^^^^
Create a DN.

\--sock-addr (string, required)
  The socket address of the DN. The socket address will be used to
  send GRPC to the DN. It is also used as the unique identifier of the
  DN. E.g. ``localhost:9720`` , ``192.168.0.10:9720``. 
\--description (string, optional)
  A string to describe the DN.
\--tr-type (string, required)
  NVMe-oF target trtype: rdma or tcp
\--adr-fam (string, required)
  NVMe-oF target adrfam: ipv4, ipv6, ib, fc, intra_host
\--tr-addr (string, required)
  NVMe-oF target address: ip or BDF
\--tr-svc-id (string, required)
  NVMe-oF target trsvcid: port number
\--location (string, optional)
  A string to indicate the location of the DN. When the VDA
  controlplane allocates multiple DNs for for a :ref:`DA <da-label>`, it
  will make sure no two DNs have the same location. An empty location
  means it can go together with any DN.
\--is-offline (boolean, optional)
  If set to true, the VDA controlplane will not allocate :ref:`PD <pd-label>`
  from this DN. By default, it is false.
\--hash-code (integer, optional)
  An integer number from 1 to 65535. If there are multiple :ref:`monitors <monitor-label>`,
  each monitor will work with a subset of the hash code. E.g., if
  there are two monitors, one monitor will communicate the DNs which
  hash-code are 1 - 32767, another monitor will communicate
  the DNs which hash-code are 32768 - 65535. If set hash-code to 0 or
  omit it, a random hash-code will be used.

vda_cli dn delete
^^^^^^^^^^^^^^^^^
Delete a DN.

\--sock-addr (string, required)
  The socket address of the DN.

vda_cli dn modify
^^^^^^^^^^^^^^^^^
Modify a DN.

\--sock-addr (string, required)
  The socket address of the DN.
\--key (string, required)
  The name of the attribute to modify. Currently only 3 attributes are
  suported: "description", "isOffline", "hashCode".
\--value (string, required)
  The new value of the attribute.

vda_cli dn list
^^^^^^^^^^^^^^^
List the DNs in the VDA cluster.

\--limit (integer, optional)
  The max number of DNs to return. The default value is 100.
\--token (string, optional)
  A strinig returned from the previous ``vda_cli dn list``
  command. Then the current command will return DNs after the last DN
  of the previous command. If it is not provided, this command will
  return from the first DN.

vda_cli dn get
^^^^^^^^^^^^^^
Get the information of a DN.

\--sock-addr (string, required)
  The socket address of the DN.

vda_cli pd
----------
The ``vda_cli pd`` sub-command is used to manage the :ref:`PD <pd-label>`.

vda_cli pd create
^^^^^^^^^^^^^^^^^
Create a PD

\--sock-addr (string, required)
  The socket address of the DN.
\--pd-name (string, required)
  The PD name, it should be unique across the DN.
\--description (string, optional)
  A string to describe the PD.
\--is-offline (boolean, optional)
  If set to true, the VDA controlplane will not allocate this PD. By
  default, it is false.
\--rw-ios-per-sec (integer, optional)
  The limit of read/write IOs per second. The default value is 0,
  which means no limit.
\--rw-mbytes-per-sec (integer, optional)
  The limit of read/write MB per second. The default value is 0, which
  means no limit.
\--r-mbytes-per-sec (integer, optional)
  The limit of read MB per second. The default value is 0, which means
  no limit.
\--w-mbytes-per-sec (integer, optional)
  The limit of write MB per second. The default value is 0, which
  means no limit.
\--bdev-type-key (integer, required)
  The type of this PD, three types are supported: malloc, aio, nvme
\--bdev-type-value (integer, required)
  If the type is malloc, this value means the size in MB of the malloc
  bdev. If the type is aio, this value means the file or device path
  of the aio bdev. If the type is nvme, this value means the PCIE
  address of the nvme bdev.

vda_cli pd delete
^^^^^^^^^^^^^^^^^
Delete a PD

\--sock-addr (string, required)
  The socket address of the DN.
\--pd-name (string, required)
  The PD name in the DN.

vda_cli pd modify
^^^^^^^^^^^^^^^^^
Modify a PD

\--sock-addr (string, required)
  The socket address of the DN.
\--pd-name (string, required)
  The PD name in the DN.
\--key (string, required)
  The name of the attribute to modify. Two attributes are supported:
  "description", "isOffline".
\--value (string, required)
  The new value of the attribute.

vda_cli pd list
^^^^^^^^^^^^^^^
List the PDs in a specific DN.

\--sock-addr (string, required)
  The socket address of the DN.

vda_cli pd get
^^^^^^^^^^^^^^
Get the information of a PD

\--sock-addr (string, required)
  The socket address of the DN.
\--pd-name (string, required)
  The PD name in the DN.

vda_cli cn
----------
The ``vda_cli cn`` sub-command is used to manage the :ref:`CN <cn-label>`.

vda_cli cn create
^^^^^^^^^^^^^^^^^
Create a CN
\--sock-addr (string)
  The  socket address of the CN. The socket address will be used to
  send GRPC to the CN. It is also used as the unique identifier of the
  CN. E.g. ``localhost:9820``, ``192.168.0.20:9820``.
\--description (string)
  A string to describe the CN.
\--tr-type (string)
  NVMe-oF target trtype: rdma or tcp
\--adr-fam (string)
  NVMe-oF target adrfam: ipv4, ipv6, ib, fc, intra_host
\--tr-addr (string)
  NVMe-oF target address: ip or BDF
\--tr-svc-id (string)
  NVMe-oF target trsvcid: port number
\--location (string)
  A string to indicate the location of the Cn. When the VDA
  controlplane allocates multiple CNs for for a :ref:`DA <da-label>`, it
  will make sure no two CNs have the same location. An empty location
  means it can go together with any CN.
\--is-offline (boolean)
  If set to true, the VDA controlplane will not allocate :ref:`cntlr <cntlr-label>`
  from this CN. By default, it is false.
\--hash-code (integer)
  An integer number from 1 to 65535. If there are multiple :ref:`monitors <monitor-label>`,
  each monitor will work with a subset of the hash code. E.g., if
  there are two monitors, one monitor will communicate the CNs which
  hash-code are 1 - 32767, another monitor will communicate
  the CNs which hash-code are 32768 - 65535. If set hash-code to 0 or
  omit it, a random hash-code will be used.

vda_cli cn delete
^^^^^^^^^^^^^^^^^
Delete a CN.

\--sock-addr (string)
  The socket address of the CN.

vda_cli cn modify
^^^^^^^^^^^^^^^^^
Modify a CN

\--sock-addr (string)
  The socket address of the CN
\--key (string)
  The name of the attribute to modify. Currently only 3 attributes are
  suported: "description", "isOffline", "hashCode".
\--value (string)
  The new value of the attribute.

vda_cli cn list
^^^^^^^^^^^^^^^
List the CNs in the VDA cluster.

\--limit (integer, optional)
  The max number of CNs to return.
\--token (string, optional)
  A strinig returned from the previous ``vda_cli cn list``
  command. Then the current command will return CNs after the last CN
  of the previous command. If it is not provided, this command will
  return from the first CN.

vda_cli cn get
^^^^^^^^^^^^^^
Get the information of a CN.

\--sock-addr (string, required)
  The socket address of the CN.

vda_cli da
----------
The ``vda_cli da`` sub-command is used to manage the :ref:`DA <da-label>`.

vda_cli da create
-----------------
Create a DA.

\--da-name (stirng, required)
  The name of the da. It should be unique across all DAs in the same
  VDA cluster.
\--description (stirng, optional)
  A string to describe the DN.
\--size-mb (stirng, required)
  The DA size in MB.
\--physical-size-mb (string, required)
  The physical size to allocate. Currently, please alwasy set it to
  the same value as the ``--size-mb``.
\--cntlr-cnt (string, required)
  How many :ref:`cntlrs <cntlr-label>` the DA will have.
\--strip-cnt (string, required)
  The DA is a raid0 device. This parameter indicate how many legs the
  raid0 device will have.
\--strip-size-kb (strip, required)
  The raid0 strip size in KB.
\--rw-ios-per-sec (integer, optional)
  The read/write IOs per second.
\--rw-mbytes-per-sec (integer, optional)
  The read/write MB per second.
\--r-mbytes-per-sec (integer, optional)
  The read MB per second.
\--w-mbytes-per-sec (integer, optional)
  The write MB per second.

vda_cli da delete
^^^^^^^^^^^^^^^^^
Delete a DA.

\--da-name
  The DA name.

vda_cli da modify
^^^^^^^^^^^^^^^^^
Modify a DA.

\--da-name (string, required)
  The DA name.
\--key (string, required)
  The  name of hte attribute to modify. Currently only 1 attribute is
  supported: "description".
\--value (string, required)
  The new vlaue of the attribute.

vda_cli da list
^^^^^^^^^^^^^^^
List the DAs in the VDA cluster.

\--limit (integer, optional)
  The max number of CNs to return.
\--token (string, optional)
  A strinig returned from the previous ``vda_cli da list``
  command. Then the current command will return DAs after the last DA
  of the previous command. If it is not provided, this command will
  return from the first DA.

vda_cli da get
^^^^^^^^^^^^^^
Get the information of a DA

\--da-name (string, required)
  The DA name.

vda_cli exp
-----------
The ``vda_cli exp`` sub-command is used to manage the :ref:`EXP <exp-label>`.

vda_cli exp create
^^^^^^^^^^^^^^^^^^
Create an EXP.

\--da-name (string, required)
  The DA name.
\--exp-name (string, required)
  The EXP name.
\--description (string, optional)
  A string to describe the EXP.
\--initiator-nqn (string, required)
  The initiator (:ref:`host <host-label>`) nqn. The EXP will only
  allow this initiator to access it.
\--snap-name (string, optional)
  Currently please always omit this value.

vda_cli exp delete
^^^^^^^^^^^^^^^^^^
Delete an EXP.

\--da-name (string, required)
  The DA name.
\--exp-name (string, required)
  The EXP name.

vda_cli exp modify
^^^^^^^^^^^^^^^^^^
Modify an EXP.

\--da-name (string, required)
  The DA name.
\--exp-name (string, required)
  The EXP name.
\--key (string, required)
  The name of the attribute to modify. Currently only one attribute is
  supported: "description".
\--value (string, required)
  The new value of the attribute.

vda_cli exp list
^^^^^^^^^^^^^^^^
List the EXPs in a specific DA.

\--da-name (string, required)
  The DA name.

vda_cli exp get
^^^^^^^^^^^^^^^
Get the information of an EXP.

\--da-name (string, required)
  The DA name.
\--exp-name (string, required)
  The EXP name.
