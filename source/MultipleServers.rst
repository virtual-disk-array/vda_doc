Multiple Servers Deployment
===========================

A production environment have lots of :ref:`DNs <dn-label>` and
:ref:`CNs <cn-label>`, and may have multiple :ref:`portals <portal-label>`
and  :ref:`monitors <monitor-label>` for high availability and
scalability. In this tutorial, we will deploy two instances for each
component. Below is the architecture:

.. image:: /images/multiple_servers_deployment.png

Below are the ip address of each components:

* 192.168.0.10 dn0
* 192.168.0.11 dn1
* 192.168.0.12 cn0
* 192.168.0.13 cn1
* 192.168.0.14 host0
* 192.168.0.15 host1
* 192.168.0.16 etcd
* 192.168.0.17 portal0
* 192.168.0.18 portal1
* 192.168.0.19 monitor0
* 192.168.0.20 monitor1
* 192.168.0.21 cli

Here we only deploy a single etcd server. We could deploy multiple
etcd servers, please refer the `etcd cluster guide <https://etcd.io/docs/v3.4/op-guide/clustering/>`_.
But the etcd cluster is out of the scope of this guide. So here we
only deploy a single etcd for demo.

Launch etcd
^^^^^^^^^^^
Login to the etcd server (192.168.0.16).

* Create work directory::

    mkdir -p /tmp/vda_data

* Follow the `install guide <https://etcd.io/docs/v3.4/install/>`_ to
  install etcd::

    cd ~
    curl -L -O https://github.com/etcd-io/etcd/releases/download/v3.4.16/etcd-v3.4.16-linux-amd64.tar.gz
    tar xvf etcd-v3.4.16-linux-amd64.tar.gz

* Go to the etcd directory and run below command::

    cd etcd-v3.4.16-linux-amd64
    ./etcd --listen-client-urls http://localhost:2389 \
    --advertise-client-urls http://localhost:2389 \
    --listen-peer-urls http://localhost:2390 \
    --name etcd0 --data-dir /tmp/vda_data/etcd0.data \
    > /tmp/vda_data/etcd0.log 2>&1 &

Launch dn0
^^^^^^^^^^
Login to the dn0 server (192.168.0.10).

* Create work directory::

    mkdir -p /tmp/vda_data

* Install spdk
  Follow the `SPDK Getting Started doc <https://spdk.io/doc/getting_started.html>`_.
  ::
    cd ~
    git clone https://github.com/spdk/spdk
    cd spdk
    git submodule update --init
    sudo scripts/pkgdep.sh
    ./configure
    make

* Initialize the spdk environment (run it once after every reboot)::

    sudo scripts/setup.sh

* Go to the spdk directory and launch the spdk application::

    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn.sock --wait-for-rpc > /tmp/vda_data/dn.log 2>&1 &

* Wait until the ``/tmp/vda_data/dn.sock`` is created (1 or 2 seconds
  should be enough), then run below commands::

    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock bdev_set_options -d
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_start_init
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_wait_init
    sudo chmod 777 /tmp/vda_data/dn.sock

* Install vda
  Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package.

* Go to the vda directory, launch `vda_dn_agent`::

    ./vda_dn_agent --network tcp --address '192.168.0.10:9720' \
    --sock-path /tmp/vda_data/dn.sock --sock-timeout 10 \
    --lis-conf '{"trtype":"tcp","traddr":"192.168.0.10","adrfam":"ipv4","trsvcid":"4420"}' \
    --tr-conf '{"trtype":"TCP"}' \
    > /tmp/vda_data/dn_agent.log 2>&1 &

* Get the nvme device pci address::

    lspci | grep Non-Volatile

  There are 2 nvme devices in dn0, they are: FIXME.
  You should find different pci address(es) in your environment.

Launch dn1
^^^^^^^^^^
Login to the dn1 server (192.168.0.11).

* Create work directory::

    mkdir -p /tmp/vda_data

* Install spdk
  Follow the `SPDK Getting Started doc <https://spdk.io/doc/getting_started.html>`_.
  ::
    cd ~
    git clone https://github.com/spdk/spdk
    cd spdk
    git submodule update --init
    sudo scripts/pkgdep.sh
    ./configure
    make

* Initialize the spdk environment (run it once after every reboot)::

    sudo scripts/setup.sh

* Go to the spdk directory and launch the spdk application::

    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn.sock --wait-for-rpc > /tmp/vda_data/dn.log 2>&1 &

* Wait until the ``/tmp/vda_data/dn.sock`` is created (1 or 2 seconds
  should be enough), then run below commands::

    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock bdev_set_options -d
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_start_init
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_wait_init
    sudo chmod 777 /tmp/vda_data/dn.sock

* Install vda
  Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package.

* Go to the vda directory, launch `vda_dn_agent`::

    ./vda_dn_agent --network tcp --address '192.168.0.11:9720' \
    --sock-path /tmp/vda_data/dn.sock --sock-timeout 10 \
    --lis-conf '{"trtype":"tcp","traddr":"192.168.0.11","adrfam":"ipv4","trsvcid":"4420"}' \
    --tr-conf '{"trtype":"TCP"}' \
    > /tmp/vda_data/dn_agent.log 2>&1 &

* Get the nvme device pci address::

    lspci | grep Non-Volatile

  There are 2 nvme devices in dn0, they are: FIXME.
  You should find different pci address(es) in your environment.

Launch cn0
^^^^^^^^^^
Login to the cn0 server (192.168.0.12).

* Create work directory::

    mkdir -p /tmp/vda_data

* Install spdk
  Follow the `SPDK Getting Started doc <https://spdk.io/doc/getting_started.html>`_.
  ::
    cd ~
    git clone https://github.com/spdk/spdk
    cd spdk
    git submodule update --init
    sudo scripts/pkgdep.sh
    ./configure
    make

* Initialize the spdk environment (run it once after every reboot)::

    sudo scripts/setup.sh

* Go to the spdk directory and launch the spdk application::

    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/cn.sock --wait-for-rpc > /tmp/vda_data/cn.log 2>&1 &

* Wait until the ``/tmp/vda_data/cn.sock`` is created (1 or 2 seconds
  should be enough), then run below commands::

    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock bdev_set_options -d
    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock framework_start_init
    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock framework_wait_init
    sudo chmod 777 /tmp/vda_data/cn.sock

* Install vda
  Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package.

* Go to the vda directory, launch `vda_cn_agent`::

    ./vda_cn_agent --network tcp --address '192.168.0.12:9820' \
    --sock-path /tmp/vda_data/cn.sock --sock-timeout 10 \
    --lis-conf '{"trtype":"tcp","traddr":"192.168.0.12","adrfam":"ipv4","trsvcid":"4430"}' \
    --tr-conf '{"trtype":"TCP"}' \
    > /tmp/vda_data/cn_agent.log 2>&1 &

Launch cn1
^^^^^^^^^^
Login to the cn1 server (192.168.0.13).

* Create work directory::

    mkdir -p /tmp/vda_data

* Install spdk
  Follow the `SPDK Getting Started doc <https://spdk.io/doc/getting_started.html>`_.
  ::
    cd ~
    git clone https://github.com/spdk/spdk
    cd spdk
    git submodule update --init
    sudo scripts/pkgdep.sh
    ./configure
    make

* Initialize the spdk environment (run it once after every reboot)::

    sudo scripts/setup.sh

* Go to the spdk directory and launch the spdk application::

    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/cn.sock --wait-for-rpc > /tmp/vda_data/cn.log 2>&1 &

* Wait until the ``/tmp/vda_data/cn.sock`` is created (1 or 2 seconds
  should be enough), then run below commands::

    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock bdev_set_options -d
    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock framework_start_init
    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock framework_wait_init
    sudo chmod 777 /tmp/vda_data/cn.sock

* Install vda
  Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package.

* Go to the vda directory, launch `vda_cn_agent`::

    ./vda_cn_agent --network tcp --address '192.168.0.13:9820' \
    --sock-path /tmp/vda_data/cn.sock --sock-timeout 10 \
    --lis-conf '{"trtype":"tcp","traddr":"192.168.0.13","adrfam":"ipv4","trsvcid":"4430"}' \
    --tr-conf '{"trtype":"TCP"}' \
    > /tmp/vda_data/cn_agent.log 2>&1 &

Launch portal0
^^^^^^^^^^^^^^
Login to the portal0 server (192.168.0.17).

* Create work directory::

    mkdir -p /tmp/vda_data

* Install vda
  Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package.

* Go to the vda directory, launch `vda_portal`::

    ./vda_portal --portal-address '192.168.0.17:9520' --portal-network tcp \
    --etcd-endpoints 192.168.0.16:2389 \
    > /tmp/vda_data/portal.log 2>&1 &

Launch portal1
^^^^^^^^^^^^^^
Login to the portal1 server (192.168.0.18).

* Create work directory::

    mkdir -p /tmp/vda_data

* Install vda
  Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package.

* Go to the vda directory, launch `vda_portal`::

    ./vda_portal --portal-address '192.168.0.18:9520' --portal-network tcp \
    --etcd-endpoints 192.168.0.16:2389 \
    > /tmp/vda_data/portal.log 2>&1 &

Launch monitor0
^^^^^^^^^^^^^^^
Login to the monitor0 server (192.168.0.19).

* Create work directory::

    mkdir -p /tmp/vda_data

* Install vda
  Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package.

* Go to the vda directory, launch `vda_monitor`::

    ./vda_monitor --etcd-endpoints 192.168.0.16:2389 \
    > /tmp/vda_data/monitor.log 2>&1 &

Launch monitor1
^^^^^^^^^^^^^^^
Login to the monitor0 server (192.168.0.20).

* Create work directory::

    mkdir -p /tmp/vda_data

* Install vda
  Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package.

* Go to the vda directory, launch `vda_monitor`::

    ./vda_monitor --etcd-endpoints 192.168.0.16:2389 \
    > /tmp/vda_data/monitor.log 2>&1 &

Operate the VDA cluster
^^^^^^^^^^^^^^^^^^^^^^^
Login to the cli server (192.168.0.21)

* Install vda
  Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package. Then go to the vda directory.

* Create dn0::

    ./vda_cli dn create --sock-addr 192.168.0.10:9720 \
    --tr-type tcp --tr-addr 192.168.0.10 --adr-fam ipv4 --tr-svc-id 4420

* Create the first pd on dn0::

    ./vda_cli pd create --sock-addr 192.168.0.10:9720 --pd-name pd00 \
    --bdev-type-key nvme --bdev-type-value FIXME

* Create the second pd on dn0::

    ./vda_cli pd create --sock-addr 192.168.0.10:9720 --pd-name pd01 \
    --bdev-type-key nvme --bdev-type-value FIXME

* Create dn1::

    ./vda_cli dn create --sock-addr 192.168.0.11:9720 \
    --tr-type tcp --tr-addr 192.168.0.11 --adr-fam ipv4 --tr-svc-id 4420

* Create the first pd on dn1::

    ./vda_cli pd create --sock-addr 192.168.0.11:9720 --pd-name pd10 \
    --bdev-type-key nvme --bdev-type-value FIXME

* Create the second pd on dn1::

    ./vda_cli pd create --sock-addr 192.168.0.11:9720 --pd-name pd11 \
    --bdev-type-key nvme --bdev-type-value FIXME

* Create cn0::

    ./vda_cli cn create --sock-addr 192.168.0.12:9820 \
    --tr-type tcp --tr-addr 192.168.0.12 --adr-fam ipv4 --tr-svc-id 4430

* Create cn1::

    ./vda_cli cn create --sock-addr 192.168.0.13:9820 \
    --tr-type tcp --tr-addr 192.168.0.13 --adr-fam ipv4 --tr-svc-id 4430

* Create dn0::

    ./vda_cli da create --da-name da0 --size-mb 512 --physical-size-mb 512 \
    --cntlr-cnt 2 --strip-cnt 2 --strip-size-kb 64

* Export dn0 to host0::

    ./vda_cli exp create --da-name da0 --exp-name exp0a \
    --initiator-nqn nqn.2016-06.io.spdk:host0

* Get the NVMeOF information of exp0a::

    ./vda_cli exp get --da-name da0 --exp-name exp0a

  FIXME: exp get output

* Create dn1::

    ./vda_cli da create --da-name da1 --size-mb 1024 --physical-size-mb 1024 \
    --cntlr-cnt 2 --strip-cnt 2 --strip-size-kb 64

* Export da1 to host1::

    ./vda_cli exp create --da-name da1 --exp-name exp1a \
    --initiator-nqn nqn.2016-06.io.spdk:host1

* Get the NVMeOF information of exp1a::

    ./vda_cli exp get --da-name da1 --exp-name exp1a

  FIXME: exp get output

Connect to da0/exp0a from host0
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Login to host0 (192.168.0.14)

* Make sure nvme-tcp kernel module is inserted::

    sudo modprobe nvme-tcp

* Make sure nvme-cli is installed, e.g. on ubutun system::

    sudo apt install -y nvme-cli

* Connect to the two cntlrs of dn0/exp0a::

    FIXME

* access the dn0/exp0a::

    FIXME

* disconnect from dn0/exp0a::

    FIXME

Connect to da1/exp1a from host1
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Login to host1 (192.168.0.15)

* Make sure nvme-tcp kernel module is inserted::

    sudo modprobe nvme-tcp

* Make sure nvme-cli is installed, e.g. on ubutun system::

    sudo apt install -y nvme-cli

* Connect to the two cntlrs of dn0/exp0a::

    FIXME

* access the dn0/exp0a::

    FIXME

* disconnect from dn0/exp0a::

    FIXME

Export dn0 to host1
^^^^^^^^^^^^^^^^^^^
Login to the cli server (192.168.0.21)

* Delete the dn0/exp0a

* Export dn0 to host1

* Get the dn0/exp0b NVMeOF information

Connect to da0/exp0b from host1
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Login to host1 (192.168.0.15)

Cleanup the environment
^^^^^^^^^^^^^^^^^^^^^^^
* Login to the cli server (192.168.0.21), run below commands

* Login to dn0, dn1, cn0, cn1, run below commands

* Login to 
