dn agent configuration
======================

command line parameters
-----------------------

\--listener
^^^^^^^^^^^
The grpc listen address. You may use 0.0.0.0 to let it listen on all
address. Or use 127.0.0.1 to listen on the local address. Or a
specific ip address to let it only listen on that address,
e.g. 182.168.0.10. The default value is 127.0.0.1.

\--port
^^^^^^^
The grpc listen port. The default value is 9720.

\--max-workers
^^^^^^^^^^^^^^
It indicate how many concurrent workers the grpc server will have. The
default value is 10.

\--sock-path
^^^^^^^^^^^^
The spdk application socket path. The default value is
/var/tmp/spdk.sock, which is the spdk application default sock path
too.

\--sock-timeout
^^^^^^^^^^^^^^^
The timeout (in second) between the dn_agent and the spdk
application. The default value is 60.

\--transport-conf
^^^^^^^^^^^^^^^^^
This parameter is a json string. It will be passed to the
nvmf_create_transport API of spdk. The default value is
'{"trtype":"TCP"}',  which means it will create a TCP transport by
default.

\--listener-conf
^^^^^^^^^^^^^^^^
This parameter is a json string. It will be passed to the
nvmf_subsystem_add_listener API of spdk.The default value is
'{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}'

\--local-store
^^^^^^^^^^^^^^
A local file path to store the configuration data of this node. The
VDA cluster sync up configuration data from the database to all the
disk nodes and controller nodes. Each node only gets the configuration
data related to itself. If set this parameter, this node will store
the configuration data to a local file. When it restarted next time,
it can read from the local file. Then it will be available before the
configuratino data is sent from the database to it.
