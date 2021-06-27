portal configuration
====================
The portal is a GRPC server. It is used to create/modify/delete all
the VDA resources. When it receives a GRPC call, it stores the data to
etcd and syncup the data to each related :ref:`DN <dn-label>` and each
:ref:`CN <cn-label>`

command line parameters
-----------------------

--etcd-endpoints
  The etcd endpoint list, splited by comma. E.g. ``localhost:2379``,
  ``192.168.0.10:2379,192.168.0.11:2379,192.168.0.12:2379``. The default
  value is localhost:2379.

--portal-network
  It will be used as the ``network`` parameter of the golang
  `net.Listen <https://golang.org/pkg/net/#Listen>`_ function. The
  allowed values are "tcp", "tcp4", "tcp6", "unix" or "unixpacket". The
  default value is "tcp".

--portal-address
  It will be used as the ``address`` parameter of the golang
  `net.Listen <https://golang.org/pkg/net/#Listen>`_ function. The
  default value is :9520.

examples
--------

* Listen on the tcp 127.0.0.1:9520, the etcd address is localhost:2379::

    vda_portal --portal-network tcp --portal-address 127.0.0.1:9520 --etcd-endpoint localhost:2379

* Listen on the all address of tcp port 9520, the etcd cluster addresses are
  192.168.0.10:2379, 192.168.0.11:2379, 192.168.0.12:2379::

    vda_portal --portal-network tcp --portal-address :9520 --etcd-endpoint 192.168.0.10:2379,192.168.0.11:2379,192.168.0.12:2379

