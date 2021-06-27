cn agent configuration
======================
The cn agent is a GRPC server. It runs on the :ref:`CN <dn-label>`. It
receives the DN configuration from :ref:`portal <portal-label>` and
:ref:`monitor <monitor-labe>`, then apply the configuration to the
spdk application.

command line parameters
-----------------------

--network
  It will be used as the ``network`` parameter of the golang
  `net.Listen <https://golang.org/pkg/net/#Listen>`_ function. The
  allowed values are "tcp", "tcp4", "tcp6", "unix" or "unixpacket". The
  default value is "tcp".

--address
  It will be used as the ``address`` parameter of the golang
  `net.Listen <https://golang.org/pkg/net/#Listen>`_ function. The
  default value is :9820.

--sock-path
  The spdk application socket path. When you launch the spdk
  application, you may set the socket path var the ``--rpc-socket``
  parameter. Then you should provide the same path here. The dn agent
  will communicate with the spdk application via the socket path. The
  default value is "/var/tmp/spdk.sock", which is also the default value
  of the spdk application. For more details about the spdk application
  ``-rpc-socket`` parameter, please refer
  https://spdk.io/doc/app_overview.html .

--sock-timeout
  The timeout in second when communicate with the spdk application. The
  default value is 10.

--tr-conf
  This is a json string, will be passed to the spdk
  ``nvmf_create_transport`` rpc during the cn agent initialize
  stage.  Please refer the ``nvmf_create_transport`` rpc in the
  `spdk rpc document <https://spdk.io/doc/jsonrpc.html>` for more
  details. The default value is '{"trtype":"TCP"}'.


--lis-conf
  This is a json string, When the cn agent creates a :ref:`EXP <exp-label>`
  this json string will be passed to the ``listen_address``
  parameter of the spdk ``nvmf_subsystem_add_listener`` rpc. Please
  refer the ``nvmf_subsystem_add_listener`` rpc in the
  `spdk rpc document <https://spdk.io/doc/jsonrpc.html>` for more
  details. The default value is
  '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}'.

examples
--------

* Let the cn agent listen on tcp port 9820. Set TCP transport max
  queue size to 64, and let the NVMeOF listen on the ip address
  192.168.0.21::

    vda_cn_agent --network tcp --address ':9820' --tr-conf '{"trtype":"TCP","max_queue_depth":64}' --lis-conf '{"trtype":"tcp","traddr":"192.168.0.21","adrfam":"ipv4","trsvcid":"4430"}'
