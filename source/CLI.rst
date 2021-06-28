CLI
===
The vda_cli exportal all the
`portal grpc <https://github.com/virtual-disk-array/vda/blob/master/pkg/proto/portalapi/portalapi.proto>`_
transparently.

dn sub-commands
---------------
The dn sub-commands are used to manage the :ref:`DN <dn-label>`.

dn create
^^^^^^^^^
Create a DN.

--sock-addr
  The socket address of the DN. The socket address will be used to
  send GRPC to the DN. It is also used as the unique identifier of the
  DN. E.g. ``localhost:9720`` , ``192.168.0.10:9720``. 
--description
  A string to describe the DN.
--tr-type
  NVMe-oF target trtype: rdma or tcp
--adr-fam
  NVMe-oF target adrfam: ipv4, ipv6, ib, fc, intra_host
--tr-addr
  NVMe-oF target address: ip or BDF
--tr-svc-id
  NVMe-oF target trsvcid: port number
--location
  A string to indicate the location of the DN. When the VDA
  controlplane allocates multiple DNs for for a :ref:`DA <da-label>`, it
  will make sure no two DNs have the same location. An empty location
  means it can go together with any DN.
--is-offline
  If set to true, the VDA controlplane will not allocate :ref:`PD <pd-label>`
  from this DN. By default, it is false.
--hash-code
  An integer number from 1 to 65535. If there are multiple :ref:`monitors <monitor-label>`,
  each monitor will work with a subset of the hash code. E.g., if
  there are two monitors, one monitor will communicate the DNs which
  hash-code are 1 - 32767, another monitor will communicate
  the DNs which hash-code are 32768 - 65535. If set hash-code to 0 or
  omit it, a random hash-code will be used.

dn delete
^^^^^^^^^
Delete a DN.

--sock-addr
  The socket address of the DN.

dn modify
^^^^^^^^^
Modify a DN

--sock-addr
  The socket address of the DN.
--key
  The name of attribute to modify. Currently only 3 options are
  suported: "description", "isOffline", "hashCode".
--value
  The value of the attribute to modify.

dn list
^^^^^^^
List the DNs in the VDA cluster.

--limit
  The max number of DNs to return.
--token
  A strinig returned from the previous ``vda_cli dn list``
  command. Then the current command will return DNs after the last DN
  of the previous command.

dn get
^^^^^^
Get the information of a DN.

--sock-addr
  The socket address of the DN.

cn sub-commands
---------------
The  cn sub-commands are used to manage the :ref:`CN <cn-label>`.

cn create
^^^^^^^^^
Create a CN
--sock-addr
  The  socket address of the CN. The socket address will be used to
  send GRPC to the CN. It is also used as the unique identifier of the
  CN. E.g. ``localhost:9820``, ``192.168.0.20:9820``.
--description
  A string to describe the CN.
--tr-type
  NVMe-oF target trtype: rdma or tcp
--adr-fam
  NVMe-oF target adrfam: ipv4, ipv6, ib, fc, intra_host
--tr-addr
  NVMe-oF target address: ip or BDF
--tr-svc-id
  NVMe-oF target trsvcid: port number
--location
  A string to indicate the location of the Cn. When the VDA
  controlplane allocates multiple CNs for for a :ref:`DA <da-label>`, it
  will make sure no two CNs have the same location. An empty location
  means it can go together with any CN.
--is-offline
  If set to true, the VDA controlplane will not allocate :ref:`cntlr <cntlr-label>`
  from this CN. By default, it is false.
--hash-code
  An integer number from 1 to 65535. If there are multiple :ref:`monitors <monitor-label>`,
  each monitor will work with a subset of the hash code. E.g., if
  there are two monitors, one monitor will communicate the CNs which
  hash-code are 1 - 32767, another monitor will communicate
  the CNs which hash-code are 32768 - 65535. If set hash-code to 0 or
  omit it, a random hash-code will be used.

cn delete
^^^^^^^^^
Delete a CN.

--sock-addr
  The socket address of the CN.

cn modify
^^^^^^^^^
Modify a CN

--sock-addr
  The socket address of the CN
--key
  The name of attribute to modify. Currently only 3 options are
  suported: "description", "isOffline", "hashCode".
--value
  The value of the attribute to modify

cn list
^^^^^^^
List the CNs in the VDA cluster.

--limit
  The max number of CNs to return.
--token
  A strinig returned from the previous ``vda_cli cn list``
  command. Then the current command will return Cns after the last CN
  of the previous command.

cn get
^^^^^^
Get the information of a CN.

--sock-addr
  The socket address of the CN.
