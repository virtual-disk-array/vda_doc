Multiple Components In One Server
=================================

The VDA cluster could manager thousands of disk nodes and controller
nodes. For testing purpose, we could put multiple disk nodes and
controller nodes to a single server. This guide will show you how to
manage multiple disk nodes and controller nodes. All of the nodes and
the controller plane components are in the same server. Below is the
architecture:

.. image:: /images/multiple_components_in_one_server.png

Each cn_agent and dn_agent should have its own spdk application. We
will deploy 2 cn_agent and 2 dn_agent, so we will launch 4 spdk
applications. All of the should listen on different port. We only have
one database, one portal and one monitor. You could deploy multiple
portals and put them to a load balancer. And you could launch multiple
monitors too.

We deploy these components to a ubuntu20.04 server, but they should be
deployed to most of the linux distributions. To deploy multiple spdk
applications, the server should have at least 16G memory, and we will
allocate 8G huge page for spdk applications.

Install spdk
^^^^^^^^^^^^
Follow the `SPDK Getting Started doc <https://spdk.io/doc/getting_started.html>`_.

.. code-block:: none

   cd ~
   git clone https://github.com/spdk/spdk
   cd spdk
   git submodule update --init
   sudo scripts/pkgdep.sh
   ./configure
   make

Run spdk
^^^^^^^^
We will have 2 controller nodes and two disk nodes. So we will launch
4 spdk applications. And we should disable auto examine on all of
them. Before launch any spdk application, we should initialize the
spdk environment. Please run below commands under the spdk directory:

.. code-block:: none

   sudo HUGEMEM=8192 scripts/setup.sh

Please note, if you don't hvae 8192M huge page, please set
HUGEMEM=5000 at least. The 4096M may not work for 4 spdk applications.

Launch the dn_spdk_app_0

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/dn0.sock --wait-for-rpc > /tmp/dn0.log 2>&1 &

Wait until the /tmp/dn0.sock is created, then run below commands:

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/dn0.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/dn0.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/dn0.sock framework_wait_init
   sudo chmod 777 /tmp/dn0.sock

Launch the dn_spdk_app_1

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/dn1.sock --wait-for-rpc > /tmp/dn1.log 2>&1 &

Wait until the /tmp/dn1.sock is created, then run below commands:
.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/dn1.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/dn1.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/dn1.sock framework_wait_init
   sudo chmod 777 /tmp/dn1.sock

Launch the cn_spdk_app_0

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/cn0.sock --wait-for-rpc > /tmp/cn0.log 2>&1 &

Wait until the /tmp/cn0.sock is created, then run below commands:
.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/cn0.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/cn0.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/cn0.sock framework_wait_init
   sudo chmod 777 /tmp/cn0.sock

launch the cn_spdk_app_1

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/cn1.sock --wait-for-rpc > /tmp/cn1.log 2>&1 &

Wait until the /tmp/cn1.sock is created, then run below commands:
.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/cn1.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/cn1.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/cn1.sock framework_wait_init
   sudo chmod 777 /tmp/cn1.sock

Install vda
^^^^^^^^^^^
Install venv, create a python virtual environment, install vda in this
environment.

.. code-block:: none

   cd ~/
   sudo apt install -y python3-venv
   python3 -m venv vda_env
   source vda_env/bin/activate
   pip install vda

All of below commands should be invoked under the vda_env. If you run
below commands in a new terminal, make sure run below command to come
into the vda_env:

.. code-block:: none

   soruce vda_env/bin/activate

Init database
^^^^^^^^^^^^^

.. code-block:: none

   vda_db --action create --db-uri sqlite:////tmp/vda.db

Launch portal
^^^^^^^^^^^^^

.. code-block:: none

   nohup vda_portal --listener 127.0.0.1 --port 9520 --db-uri sqlite:////tmp/vda.db > /tmp/vda_portal_0.log 2>&1 &

Launch monitor
^^^^^^^^^^^^^^

.. code-block:: none

   nohup vda_monitor --listener 127.0.0.1 --port 9620 --db-uri sqlite:////tmp/vda.db > /tmp/vda_monitor_0.log 2>&1 &

Launch two dn_agents
^^^^^^^^^^^^^^^^^^^^

.. code-block:: none

   nohup vda_dn_agent --listener 127.0.0.1 --port 9720 --sock-path /tmp/dn0.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}' > /tmp/vda_dn_agent_0.log 2>&1 &
   nohup vda_dn_agent --listener 127.0.0.1 --port 9721 --sock-path /tmp/dn1.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4421"}' > /tmp/vda_dn_agent_1.log 2>&1 &

We launch two disk nodes on the same server, so we should let the two
nodes listen on different ports. The dn0 listens on 9720 for the gRPC,
and listens on 4420 for the TCP NVMeOF. The dn1 listens on 9721 for
the gRPC, and listens on 4421 for the TCP NVMeOF.

Launch two cn_agents
^^^^^^^^^^^^^^^^^^^^

.. code-block:: none

   nohup vda_cn_agent --listener 127.0.0.1 --port 9820 --sock-path /tmp/cn0.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}' > /tmp/vda_cn_agent_0.log 2>&1 &
   nohup vda_cn_agent --listener 127.0.0.1 --port 9821 --sock-path /tmp/cn1.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4431"}' > /tmp/vda_cn_agent_1.log 2>&1 &

Similar as disk nodes, the two controller nodes should listen on
different ports. The cn0 listens on 9820 for the gRPC, and listens on
4430 for the TCP NVMeOF. The cn1 listens on 9821 for the gRPC, and
listens on 4431 for the TCP NVMeOF.

Operate against the cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Run below commands to add the two disk nodes to the cluster:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 dn create --dn-name localhost:9720 --dn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}' --location localhost:9720
   vda_cli --addr-port 127.0.0.1:9520 dn create --dn-name localhost:9721 --dn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4421"}' --location localhost:9721

The --location is a string, VDA will make sure a disk array allocate
disks from different locations. We set different locations for the two
disk nodes. When we create a disk array, then the physical disks
of the disk array will be across different nodes.

Create 4 files, we use them as physical disks:

.. code-block:: none

   dd if=/dev/zero of=/tmp/a.img bs=1M count=256
   dd if=/dev/zero of=/tmp/b.img bs=1M count=256
   dd if=/dev/zero of=/tmp/c.img bs=1M count=256
   dd if=/dev/zero of=/tmp/d.img bs=1M count=256

Add the four physical disks to the two disk nodes, each node has two
disks:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9720 --pd-name pd0 --pd-conf '{"type":"aio","filename":"/tmp/a.img"}'
   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9720 --pd-name pd1 --pd-conf '{"type":"aio","filename":"/tmp/b.img"}'
   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9721 --pd-name pd0 --pd-conf '{"type":"aio","filename":"/tmp/c.img"}'
   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9721 --pd-name pd1 --pd-conf '{"type":"aio","filename":"/tmp/d.img"}'

The physical disks in the same disk node should have different
names. But they could have the same name if they are in different disk
nodes.

Create two controller nodes

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 cn create --cn-name localhost:9820 --cn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}' --location localhost:9820
   vda_cli --addr-port 127.0.0.1:9520 cn create --cn-name localhost:9821 --cn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4431"}' --location localhost:9821

Similar as disk nodes, we set different locations for the two
controller nodes.

Create a disk array

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 da create --da-name da0 --cntlr-cnt 2 --da-size 33554432 --physical-size 33554432 --da-conf '{"stripe_count":2, "stripe_size_kb":64}'

'--da-name da0' means the disk array name is da0, it should be a
unique name in the cluster.

'--cntlr-cnt 2' means the disk array da0 has two controllers.

'stripe_count' is 2 means the raid0 of da0 has two legs.

Export the disk array da0 to localhost

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 exp create --da-name da0 --exp-name exp0 --initiator-nqn nqn.2016-06.io.spdk:host0

Get the exportor status:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 exp get --da-name da0 --exp-name exp0

The output should be similar as below:

.. code-block:: none

   {
     "reply_info": {
       "req_id": "1fef27c989854eb5afb1265454a1b0c2",
       "reply_code": 0,
       "reply_msg": "success"
     },
     "exp_msg": {
       "exp_id": "fb210bb7b8434eba89cd11c4b66711af",
       "exp_name": "exp0",
       "exp_nqn": "nqn.2016-06.io.spdk:vda-exp-da0-exp0",
       "da_name": "da0",
       "initiator_nqn": "nqn.2016-06.io.spdk:host0",
       "snap_name": "",
       "es_msg_list": [
         {
           "es_id": "2e58f7cc7ad7488589f05c6645145b82",
           "cntlr_idx": 0,
           "cn_name": "localhost:9820",
           "cn_listener_conf": "{\"trtype\":\"tcp\",\"traddr\":\"127.0.0.1\",\"adrfam\":\"ipv4\",\"trsvcid\":\"4430\"}",
           "error": false,
           "error_msg": ""
         },
         {
           "es_id": "f2e97522364c4175900fba50fef80d80",
           "cntlr_idx": 1,
           "cn_name": "localhost:9821",
           "cn_listener_conf": "{\"trtype\":\"tcp\",\"traddr\":\"127.0.0.1\",\"adrfam\":\"ipv4\",\"trsvcid\":\"4431\"}",
           "error": false,
           "error_msg": ""
         }
       ]
     }
   }


We can find the connection information we need from the output. The
"exp_nqn" is the NQN of the disk array. The es_msg_list has two items,
they are the information of the two controllers. We need the two
cn_listener_conf when we connect to the two controllers.

Before connect to the disk array, make sure nvme-tcp module is loaded,
nvme-cli and jq are inistalled:

.. code-block:: none

   sudo modprobe nvme-tcp
   sudo apt install -y nvme-cli
   sudo apt inistall -y jq

Connect to the two controllers:

.. code-block:: none

   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da0-exp0 -a 127.0.0.1 -s 4430 --hostnqn nqn.2016-06.io.spdk:host0
   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da0-exp0 -a 127.0.0.1 -s 4431 --hostnqn nqn.2016-06.io.spdk:host0

Find the nvme devices of the two controllers:

.. code-block:: none

   sudo nvme list-subsys -o json | jq '.Subsystems[] | select(.NQN=="nqn.2016-06.io.spdk:vda-exp-da0-exp0")'

The output should be something like below:

.. code-block:: none

   {
     "Name": "nvme-subsys1",
     "NQN": "nqn.2016-06.io.spdk:vda-exp-da0-exp0",
     "Paths": [
       {
         "Name": "nvme1",
         "Transport": "tcp",
         "Address": "traddr=127.0.0.1 trsvcid=4430",
         "State": "live"
       },
       {
         "Name": "nvme2",
         "Transport": "tcp",
         "Address": "traddr=127.0.0.1 trsvcid=4431",
         "State": "live"
       }
     ]
   }


If the "CONFIG_NVME_MULTIPATH" is enabled in the linux kernel, linux
kernel will combine the two controllers to a single device. When you
access /dev/nvme1n1 , the traffic will be distributed to both nvme1
and nvme2, and if one controller is failed, kernel will failover
automatically.

Clean up all resources
^^^^^^^^^^^^^^^^^^^^^^

Disconnect the disk array

.. code-block:: none

   sudo nvme disconnect -n nqn.2016-06.io.spdk:vda-exp-da0-exp0

The output should be something like below:

.. code-block:: none

   NQN:nqn.2016-06.io.spdk:vda-exp-da0-exp0 disconnected 2 controller(s)

You can find it disconnected from 2 controllers.

Delete the exporter

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 exp delete --da-name da0 --exp-name exp0

Delete the disk array

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 da delete --da-name da0

Delete the two controller nodes

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 cn delete --cn-name localhost:9820
   vda_cli --addr-port 127.0.0.1:9520 cn delete --cn-name localhost:9821

Delete the physical disks

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 pd delete --dn-name localhost:9720 --pd-name pd0
   vda_cli --addr-port 127.0.0.1:9520 pd delete --dn-name localhost:9720 --pd-name pd1
   vda_cli --addr-port 127.0.0.1:9520 pd delete --dn-name localhost:9721 --pd-name pd0
   vda_cli --addr-port 127.0.0.1:9520 pd delete --dn-name localhost:9721 --pd-name pd1

Delete the two disk nodes

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 dn delete --dn-name localhost:9720
   vda_cli --addr-port 127.0.0.1:9520 dn delete --dn-name localhost:9721

Kill all processes

.. code-block:: none

   killall vda_portal
   killall vda_monitor
   killall vda_dn_agent
   killall vda_cn_agent
   sudo killall reactor_0

Drop the database

.. code-block:: none

   vda_db --action drop --db-uri sqlite:////tmp/vda.db
