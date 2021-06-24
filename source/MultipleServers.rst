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

* Go to the spdk directory and run below commands::

    sudo scripts/setup.sh
    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn.sock --wait-for-rpc > /tmp/vda_data/dn.log 2>&1 &
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

* Go to the spdk directory and run below commands::

    sudo scripts/setup.sh
    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn.sock --wait-for-rpc > /tmp/vda_data/dn.log 2>&1 &
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

* Go to the spdk directory and run below commands::

    sudo scripts/setup.sh
    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn.sock --wait-for-rpc > /tmp/vda_data/dn.log 2>&1 &
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock bdev_set_options -d
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_start_init
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_wait_init
    sudo chmod 777 /tmp/vda_data/dn.sock

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

* Go to the spdk directory and run below commands::

    sudo scripts/setup.sh
    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn.sock --wait-for-rpc > /tmp/vda_data/dn.log 2>&1 &
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock bdev_set_options -d
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_start_init
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_wait_init
    sudo chmod 777 /tmp/vda_data/dn.sock

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





