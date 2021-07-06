Work With Kubernetes
====================
The VDA can be configured as the kubernetes
`persistent volume <https://kubernetes.io/docs/concepts/storage/persistent-volumes/>`_
via `CSI interface <https://kubernetes.io/docs/concepts/storage/volumes/#csi>`_.
This guide creates a minimal kubernetes cluster via minikube, and
creates a minimal vda cluster, then let the kubernetes cluster claim
persistent volume(s) from the vda cluster. Below are the components we
will deploy:

.. image:: /images/vda_and_minikube.png

We deploay the kubernetes cluster to a single server, and deploy all
the vda components to another server.

* kubernetes cluster server ip: 192.168.1.30
* vda cluster server ip: 192.168.1.31

Deploy VDA
----------
The steps are similar as the :ref:`Minimal Deployment <minimal-deployment-label>`.
But there is an important different: In the :ref:`Minimal Deployment <minimal-deployment-label>`,
all the vda components allow local connections (connect from localhost
or 127.0.0.1) only. In this guide, the vda cluster should allow two
kind of exteranl connections from the kubernetes server:

* the gRPC to :ref:`portal <portal-label>`,
  for creating/deleting :ref:`DA <da-label>` and :ref:`EXP <exp-label>`

* the NVMeOF to :ref:`cntlr <cntlr-label>`, for attaching/detaching volume(s)

For simplify the steps, we let all the VDA compoents allow exteranl
connections. Login to the vda cluster server (1291.68.1.30), then
perform below actions:

Prepare
^^^^^^^
* Create a work directory::

    mkdir -p /tmp/vda_data

* Install etcd::

    curl -L -O https://github.com/etcd-io/etcd/releases/download/v3.4.16/etcd-v3.4.16-linux-amd64.tar.gz
    tar xvf etcd-v3.4.16-linux-amd64.tar.gz

* Install spdk, follow the `SPDK Getting Started doc <https://spdk.io/doc/getting_started.html>`_::

    cd ~
    git clone https://github.com/spdk/spdk
    cd spdk
    git submodule update --init
    sudo scripts/pkgdep.sh
    ./configure
    make

* Initialize the spdk environment (run it once after every reboot)::

    sudo HUGEMEM=8192 scripts/setup.sh

* Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
  Download and unzip the package.

Launch etcd
^^^^^^^^^^^
Go to the etcd directory and run below commands::

    ./etcd --listen-client-urls http://192.168.1.30:2389 \
    --advertise-client-urls http://192.168.1.30:2389 \
    --listen-peer-urls http://localhost:2390 \
    --name etcd0 --data-dir /tmp/vda_data/etcd0.data \
    > /tmp/vda_data/etcd0.log 2>&1 &

Launch :ref:`DN <dn-label>`
^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Go to the spdk directory

* Run the spdk application::

    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn.sock --wait-for-rpc > /tmp/vda_data/dn.log 2>&1 &

* Wait until the ``/tmp/vda_data/dn.sock`` is created, then run below commands::

    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock bdev_set_options -d
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_start_init
    sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_wait_init
    sudo chmod 777 /tmp/vda_data/dn.sock

* Go to the vda directory, run below command::

    ./vda_dn_agent --network tcp --address '192.168.1.30:9720' \
    --sock-path /tmp/vda_data/dn.sock --sock-timeout 10 \
    --lis-conf '{"trtype":"tcp","traddr":"192.168.1.30","adrfam":"ipv4","trsvcid":"4420"}' \
    --tr-conf '{"trtype":"TCP"}' \
    > /tmp/vda_data/dn_agent.log 2>&1 &

Launch :ref:`CN <cn-label>`
^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Go to the spdk directory

* Run the spdk application::

    sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/cn.sock --wait-for-rpc > /tmp/vda_data/cn.log 2>&1 &

* Wait until the ``/tmp/vda_data/cn.sock`` is created, then run below commands::

    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock bdev_set_options -d
    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock framework_start_init
    sudo scripts/rpc.py -s /tmp/vda_data/cn.sock framework_wait_init
    sudo chmod 777 /tmp/vda_data/cn.sock

* Go the the vda directory, run below command::

    ./vda_cn_agent --network tcp --address '192.168.1.30:9820' \
    --sock-path /tmp/vda_data/cn.sock --sock-timeout 10 \
    --lis-conf '{"trtype":"tcp","traddr":"192.168.1.30","adrfam":"ipv4","trsvcid":"4430"}' \
    --tr-conf '{"trtype":"TCP"}' \
    > /tmp/vda_data/cn_agent.log 2>&1 &

Launch :ref:`portal <portal-label>`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Go to the vda directory, run below command::

  ./vda_portal --portal-address '192.168.1.30:9520' --portal-network tcp \
   --etcd-endpoints 192.168.1.30:2389 \
   > /tmp/vda_data/portal.log 2>&1 &

Launch :ref:`monitor <monitor-label>`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Go to the vda directory, run below command::

  ./vda_monitor --etcd-endpoints 192.168.1.30:2389 \
   > /tmp/vda_data/monitor.log 2>&1 &





Deploy Kubernetes
-----------------

Create sidecars
---------------

Operate in kubernetes
---------------------

Cleanup
-------
