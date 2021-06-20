Minimal Deployment
==================

In this guide, we deploy all VDA components to a single server, and
explain the basic usage of it. Below are the components we will deploy:

.. image:: /images/minimal_deployment.png

Create a work directory
^^^^^^^^^^^^^^^^^^^^^^^^^
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

Install vda
^^^^^^^^^^^
Go to the `vda latest release <https://github.com/virtual-disk-array/vda/releases/latest>`_.
Download and unzip the package.

Launch DN components
^^^^^^^^^^^^^^^^^^^^
For each :ref:`DN <dn-label>`, we should run a spdk application and the dn_agent. First,
go to the spdk directory and run below commands::

  sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/dn.sock --wait-for-rpc > /tmp/vda_data/dn.log 2>&1 &
  sudo scripts/rpc.py -s /tmp/vda_data/dn.sock bdev_set_options -d
  sudo scripts/rpc.py -s /tmp/vda_data/dn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
  sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_start_init
  sudo scripts/rpc.py -s /tmp/vda_data/dn.sock framework_wait_init
  sudo chmod 777 /tmp/vda_data/dn.sock

We don't launch the spdk app spdk_tgt directly. We use the
``--wait-for-rpc`` option to let it stay in the init stage. Then
invoke the ``bdev_set_options`` to disable the auto examine. It is
required by the VDA. Some bdevs will be exported to :ref:`CN <cn-label>`.
So they shouldn't be examined by :ref:`DN <dn-label>`. The
``nvmf_set_crdt`` is not requried by :ref:`DN <dn-label>`, it is
required by `CN <cn-label>`. Here we invoke ``nvmf_set_crdt`` to keep
the DN and CN the same.

Then go to the vda directory and run below commands::

  ./vda_dn_agent --network tcp --address '127.0.0.1:9720' \
  --sock-path /tmp/vda_data/dn.sock --sock-timeout 10 \
  --lis-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}' \
  --tr-conf '{"trtype":"TCP"}' \
  > /tmp/vda_data/dn_agent.log 2>&1 &

The ``--lis-conf`` and ``--tr-conf`` are json strings. The values in
``--lis-conf`` will be passed to the SPDK ``nvmf_subsystem_add_listener``
RPC. The values in ``tr-conf`` will be passed to the SPDK
``nvmf_create_transport`` RPC. They are used to configure the NVMeOF
connection between :ref:`VD <vd-label>` and :ref:`cntlr <cntlr-label>`.
You can specific any values the SPDK RPCs accepts.

You can check the /tmp/vda_data/dn_agent.log, if everything is OK, you
can find below log::

  Launch dn agent server

Launch CN components
^^^^^^^^^^^^^^^^^^^^
For each :ref:`CN <cn-label>`, we should run a spdk applicaiton and the cn-agent. First,
go to the spdk directory and run below commands::

  sudo build/bin/spdk_tgt --rpc-socket /tmp/vda_data/cn.sock --wait-for-rpc > /tmp/vda_data/cn.log 2>&1 &
  sudo scripts/rpc.py -s /tmp/vda_data/cn.sock bdev_set_options -d
  sudo scripts/rpc.py -s /tmp/vda_data/cn.sock nvmf_set_crdt -t1 100 -t2 100 -t3 100
  sudo scripts/rpc.py -s /tmp/vda_data/cn.sock framework_start_init
  sudo scripts/rpc.py -s /tmp/vda_data/cn.sock framework_wait_init
  sudo chmod 777 /tmp/vda_data/cn.sock

Similar as DN, we invoke ``bdev_set_options`` to disable auto examine,
and we invoke ``nvmf_set_crdt`` to provide the delay time. The
``nvmf_set_crdt`` is requried. If we don't set it, the :ref:`cntlr <cntlr-label>`
failover may have problem.

Then go to the vda directory and run below commands::

  ./vda_cn_agent --network tcp --address '127.0.0.1:9820' \
  --sock-path /tmp/vda_data/cn.sock --sock-timeout 10 \
  --lis-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}' \
  --tr-conf '{"trtype":"TCP"}' \
  > /tmp/vda_data/cn_agent.log 2>&1 &

Similiar as DN, the ``lis-conf`` and ``--tr-conf`` are json
strings. And  the are used by the SPDK ``nvmf_subsystem_add_listener``
and ``nvmf_create_transport`` RPCs. They are used to configure the
NVMeOF connection between :ref:`cntlr <cntlr-label>` and
:ref:`host <host-label>`.

You can check the /tmp/vda_data/cn_agent.log, if everything is OK, you
can find below log::

  Launch cn agent server

Launch portal
^^^^^^^^^^^^^
Run below command::

  ./vda_portal --portal-address '127.0.0.1:9520' --portal-network tcp \
  --etcd-endpoints localhost:2389 \
  > /tmp/vda_data/portal.log 2>&1 &

We let the :ref:`portal <portal-label>` listen on the tcp 9520
port. The client should send VDA API to this port. The portal is a
stateless server, you can put mutiple portals to a load balancer.

You can check the /tmp/vda_data/portal.log, if everything is OK, you
can find below log::

  Launch portal server

Launch monitor
^^^^^^^^^^^^^^
Run below command::

  ./vda_monitor --etcd-endpoints localhost:2389 \
  > /tmp/vda_data/monitor.log 2>&1 &

By default, the monitor will send heartbeat to each CN and DN for
every 5 seconds. You can find such log message in
/tmp/vda_data/monitor.log. You can launch multiple monitors, they will
use etcd as a coordinator to split their tasks.

Create DN
^^^^^^^^^
We have launched the dn_agent, but we don't store them to the etcd
yet. So the VDA cluster doeosn't know them. We run below command to
create a :ref:`DN <dn-label>` in the VDA cluster::

  ./vda_cli dn create --sock-addr localhost:9720 --tr-svc-id 4420

Create PD
^^^^^^^^^
In this guide, we create a 256M malloc :ref:`PD <pd-label>` for demo::

  ./vda_cli pd create --sock-addr localhost:9720 --pd-name pd0 \
  --bdev-type-key malloc --bdev-type-value 256

Create CN
^^^^^^^^^
Similar as :ref:`DN <dn-label>`, we have launched the cn_agent, but
the VDA cluster doesn't know it yet. We run below command to create a
:ref:`CN <cn-label>` in the VDA cluster::

  ./vda_cli cn create --sock-addr localhost:9820 --tr-svc-id 4430

Create DA
^^^^^^^^^
We have create a :ref:`DN <dn-label>`, a :ref:`CN <cn-label>` and a
:ref:`PD <pd-label>` in the :ref:`DN <dn-label>`. Now we can create a
:ref:`DA <da-label>`. The :ref:`DA <da-label>` will allocate a
:ref:`VD <vd-label` from the `PD <pd-label>`, and allocate a
:ref:`cntlr <cntlr-label>` from the :ref:`CN <cn-label>`::

  ./vda_cli da create --da-name da0 --size-mb 64 --physical-size-mb 64 \
  --cntlr-cnt 1 --strip-cnt 1 --strip-size-kb 64

--da-name
  A unique name of the DA.
--size-mb
  The size in MegaByte of the DA
--physical-size-mb
  The sum of disk size allocated from all DNs. Currently please alwasy
  set it to the same value as "--size-mb". In the further, the VDA
  would support snapshot, the "\-\-physical-size-mb" and "\-\-size-mb"
  would be different at that time.
--cntlr-cnt
  How many :ref:`cntlr <cntlr-label>` the DA will have. We only
  created a single :ref:`CN <cn-label>`, so we can only allocate one
  cntlr.
--strip-cnt
  The raid0 strip count. If we set it to a value larger than 1, the
  VDA cluster will allocate :ref:`VD <vd-label>` from multiple
  :ref:`DN <dn-label>`. In our demo, we only have a single DN, so we
  can only set it to 1.
--strip-size-kb
  The strip size of raid0

If everything is OK, we would get below response::

  FIXME

Please note: the ``"reply_code": 0`` means the DA information has been
stored to the etcd cluster. It doesn't mean the DA has been created
successfully. To check whether the DA is created successfully, please
refer the following section.

Get DA status
^^^^^^^^^^^^^
Run below command to get the DA status::

  ./vda_cli da get --da-name da0

If everything is OK, we would get below response::

  FIXME

(FIXME: explain the result and make sure the DA has no error)

Create an EXP
^^^^^^^^^^^^^
Run below command to create an :ref:`EXP <exp-label>`::

  ./vda_cli exp create --da-name da0 --exp-name exp0a \
  --initiator-nqn nqn.2016-06.io.spdk:host0

--da-name
  The DA name
--exp-name
  The EXP name, it should be uniqu across the DA
--initiator-nqn
  The nqn of the host. The EXP will only allow this nqn connect to it.

If everything is OK, we would get below response::

  FIXME

Please note; the ``"reply_code": 0`` measn the EXP information has
been stored to the etcd cluster. It doesn't mean the EXP has been
created successfully. To check whether the EXP is created
successfully, please refer the following section.

Get EXP status
^^^^^^^^^^^^^^
Run below command to get the :ref:`EXP <exp-label>` status::

  ./vda_cli exp get --da-name da0 --exp-name exp0a

Below is the result::

  FIXME


Connect to the DA/EXP
^^^^^^^^^^^^^^^^^^^^^
Install the nvme-tcp kernel module::

  sudo modprobe nvme-tcp

Install the nvme-cli. E.g. you may run below command in a ubuntu system::

  sudo apt install -y nvme-cli

Connect to the DA/EXP::

  sudo nvme connect -t tcp -n nqn.2016-06.io.vda:exp-da0-exp0a -a 127.0.0.1 -s 4430 --hostnqn nqn.2016-06.io.spdk:host0

The disk path would be ``/dev/disk/by-id/nvme-VDA_CONTROLLER_c5e94c313982b7e362dd``.
You can use it as a normal disk on the host, e.g.::

  sudo parted /dev/disk/by-id/nvme-VDA_CONTROLLER_c5e94c313982b7e362dd print

Clean up all resources
^^^^^^^^^^^^^^^^^^^^^^
* Disconnect from the host::

    sudo nvme disconnect -n nqn.2016-06.io.vda:exp-da0-exp0a

* Delete the EXP::

    ./vda_cli exp delete --da-name da0 --exp-name exp0a

* Delete the DA::

    ./vda_cli da delete --da-name da0

* Delete the CN::

    ./vda_cli cn delete --sock-addr localhost:9820

* Delete the PD::

    ./vda_cli pd delete --sock-addr localhost:9720 --pd-name pd0

* Delete the DN::

    ./vda_cli dn delete --sock-addr localhost:9720

* Terminate all the processes::

    killall vda_portal
    killall vda_monitor
    killall vda_dn_agent
    killall vda_cn_agent
    killall etcd
    sudo killall reactor_0

* Delete the dawork directory::

    rm -rf /tmp/vda_data
