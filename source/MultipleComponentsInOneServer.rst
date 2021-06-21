Multiple Components In One Server
=================================

The VDA cluster could manager thousands of :ref:`DN <dn-label>` and
:ref:`CN <cn-label>`. Generally, each DN or CN should be deployed to a
dedicate server. But for testing purpose, we can deploy multiple DNs
and CNs to a single server. This guide will show you how to do that
and examine how VDA manages multiple DNs and CNs. Below is the
components we will deploy:

.. image:: /images/multiple_components_in_one_server.png

Each cn_agnet and dn_agent should have its own spdk application. We
will deploy 2 cn_agent and 2 dn_agent, so we will launch 4 spdk
applications. All of their NVMeOF targets should listen on different
port. To deploy these spdk applications, the server should have at
least 8G hugepages.

Create a work directory
^^^^^^^^^^^^^^^^^^^^^^^
Here we create a directory. We will store all the data (e.g. sockets,
logs, etcd data) to this directory. ::

  mkdir -p /tmp/vda_data

Install and launch etcd
^^^^^^^^^^^^^^^^^^^^^^^
Follow the `install guide <https://etcd.io/docs/v3.4/install/>`_ to
install the etcd. ::

  cd ~
  curl -L -O https://github.com/etcd-io/etcd/releases/download/v3.4.16/etcd-v3.4.16-linux-amd64.tar.gz
  tar xvf etcd-v3.4.16-linux-amd64.tar.gz

Go to the etcd directory and run below command::

  cd etcd-v3.4.16-linux-amd64
  ./etcd --listen-client-urls http://localhost:2389 \
  --advertise-client-urls http://localhost:2389 \
  --listen-peer-urls http://localhost:2390 \
  --name etcd0 --data-dir /tmp/vda_data/etcd0.data \
  > /tmp/vda_data/etcd0.log 2>&1 &

Here we don't use the default etcd port nubmers. Letter we will let
the VDA control plane components (:ref:`portal <portal-label>` and
:ref:`monitor <monitor-label>`) connect to the etcd 2398 port.

Install spdk
^^^^^^^^^^^^
Follow the `SPDK Getting Started doc <https://spdk.io/doc/getting_started.html>`_.
::

  cd ~
  git clone https://github.com/spdk/spdk
  cd spdk
  git submodule update --init
  sudo scripts/pkgdep.sh
  ./configure
  make


After build the spdk, we should setup the environment for spdk. We can
use the ``scripts/setup.sh`` in the spdk directory. Here we specific
8G hugepages::

  sudo HUGEMEM=8192 scripts/setup.sh

Install vda
^^^^^^^^^^^
Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
Download and unzip the package.

Launch dn0
^^^^^^^^^^
Go to the spdk directory, run below commands::

  sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn0.sock --wait-for-rpc > /tmp/vda_data/dn0.log 2>&1 &
  sudo scripts/rpc.py -s /tmp/vda_data/dn0.sock bdev_set_options -d
  sudo scripts/rpc.py -s /tmp/vda_data/dn0.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
  sudo scripts/rpc.py -s /tmp/vda_data/dn0.sock framework_start_init
  sudo scripts/rpc.py -s /tmp/vda_data/dn0.sock framework_wait_init
  sudo chmod 777 /tmp/vda_data/dn0.sock

Then go to the vda directory, run below commands::

  ./vda_dn_agent --network tcp --address '127.0.0.1:9720' \
  --sock-path /tmp/vda_data/dn0.sock --sock-timeout 10 \
  --lis-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}' \
  --tr-conf '{"trtype":"TCP"}' \
  > /tmp/vda_data/dn_agent_0.log 2>&1 &

Launch dn1
^^^^^^^^^^
Go to the spdk directory, run below commands::

  sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn1.sock --wait-for-rpc > /tmp/vda_data/dn1.log 2>&1 &
  sudo scripts/rpc.py -s /tmp/vda_data/dn1.sock bdev_set_options -d
  sudo scripts/rpc.py -s /tmp/vda_data/dn1.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
  sudo scripts/rpc.py -s /tmp/vda_data/dn1.sock framework_start_init
  sudo scripts/rpc.py -s /tmp/vda_data/dn1.sock framework_wait_init
  sudo chmod 777 /tmp/vda_data/dn1.sock

Then go to the vda directory, run below commands::

  ./vda_dn_agent --network tcp --address '127.0.0.1:9721' \
  --sock-path /tmp/vda_data/dn1.sock --sock-timeout 10 \
  --lis-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4421"}' \
  --tr-conf '{"trtype":"TCP"}' \
  > /tmp/vda_data/dn_agent_1.log 2>&1 &

Launch cn0
^^^^^^^^^^
Go the the spdk directory, run below commands::

  sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/cn0.sock --wait-for-rpc > /tmp/vda_data/cn0.log 2>&1 &
  sudo scripts/rpc.py -s /tmp/vda_data/cn0.sock bdev_set_options -d
  sudo scripts/rpc.py -s /tmp/vda_data/cn0.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
  sudo scripts/rpc.py -s /tmp/vda_data/cn0.sock framework_start_init
  sudo scripts/rpc.py -s /tmp/vda_data/cn0.sock framework_wait_init
  sudo chmod 777 /tmp/vda_data/cn0.sock

Then go to the vda directory, run below commands::

  ./vda_cn_agent --network tcp --address '127.0.0.1:9820' \
  --sock-path /tmp/vda_data/cn0.sock --sock-timeout 10 \
  --lis-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}' \
  --tr-conf '{"trtype":"TCP"}' \
  > /tmp/vda_data/cn_agent_0.log 2>&1 &

Launch cn1
^^^^^^^^^^
Go the the spdk directory, run below commands::

  sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/cn1.sock --wait-for-rpc > /tmp/vda_data/cn1.log 2>&1 &
  sudo scripts/rpc.py -s /tmp/vda_data/cn1.sock bdev_set_options -d
  sudo scripts/rpc.py -s /tmp/vda_data/cn1.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
  sudo scripts/rpc.py -s /tmp/vda_data/cn1.sock framework_start_init
  sudo scripts/rpc.py -s /tmp/vda_data/cn1.sock framework_wait_init
  sudo chmod 777 /tmp/vda_data/cn1.sock

Then go to the vda directory, run below commands::

  ./vda_cn_agent --network tcp --address '127.0.0.1:9821' \
  --sock-path /tmp/vda_data/cn1.sock --sock-timeout 10 \
  --lis-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4431"}' \
  --tr-conf '{"trtype":"TCP"}' \
  > /tmp/vda_data/cn_agent_1.log 2>&1 &

Launch portal
^^^^^^^^^^^^^
Run below command::

  ./vda_portal --portal-address '127.0.0.1:9520' --portal-network tcp \
  --etcd-endpoints localhost:2389 \
  > /tmp/vda_data/portal.log 2>&1 &


Launch monitor
^^^^^^^^^^^^^^
Run below command::

  ./vda_monitor --etcd-endpoints localhost:2389 \
  > /tmp/vda_data/monitor.log 2>&1 &
