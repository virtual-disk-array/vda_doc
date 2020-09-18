Minimal Deployment
==================

In this guide, we deploy all VDA components to a single service, and
describe the basic usage of it. Below is the components we will deploy:

.. image:: /images/minimal_deployment.png

The vda_cli, portal, monitor, cn_agent, dn_agent are belongs to the
VDA package. The sqlite is the database we will use. Most of linux
distributions install it by default. The cn_spdk_app and dn_spdk_app
are SPDK applications. In this tutorial, we use the buildin
application spdk_tgt. All of them are deployed in a ubuntu 20.04
server. But they should be deployed to most of the linux
distributions. In a production environment, the SPDK applications and
the VDA related processes should run as daemons. But in this guide, we
just use "nohup" command to keep them running.

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
In this tutorial, we plan to run one controller node and one disk
node. Each node should have its own spdk application. So we will
launch two spdk applications.
Before launch the spdk application, we need to initialize the spdk
environment, we could run below script in the spdk directory.

.. code-block:: none

   sudo scripts/setup.sh

When run the spdk application, we should disable auto examine, so we
should use the --wait-for-rpc parameter, and invoke an API to disable
auto examine. We set the socket path to  /tmp/dn0.sock, later we will
pass this path to the dn_agent, then the dn_agent could communicate
with this spdk application.

.. code-block:: none

   nohup sudo ./build/bin/spdk_tgt --rpc-socket /tmp/dn.sock --wait-for-rpc > /tmp/dn.log 2>&1 &

We run below command to disable auto examine:

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/dn.sock bdev_set_options -d

Then let the spdk application go to the normal mode:

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/dn.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/dn.sock framework_wait_init

Change the permission of /tmp/dn.sock, then dn_agent could access it
without root permission.

.. code-block:: none

   sudo chmod 777 /tmp/dn.sock

We do the similar things for the controller node spdk application.

Run the spdk application:

.. code-block:: none

   nohup sudo ./build/bin/spdk_tgt --rpc-socket /tmp/cn.sock --wait-for-rpc > /tmp/cn.log 2>&1 &

Wait until the spdk application is ready for accepting RPCs, then run
below commands:

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/cn.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/cn.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/cn.sock framework_wait_init
   sudo chmod 777 /tmp/cn.sock

Install vda
^^^^^^^^^^^
install venv, create a python virtual environment, install vda in this
environment.

.. code-block:: none

   cd ~
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

   nohup vda_portal --listener 127.0.0.1 --port 9520 --db-uri sqlite:////tmp/vda.db > /tmp/vda_portal.log 2>&1 &


Launch monitor
^^^^^^^^^^^^^^

.. code-block:: none

   nohup vda_monitor --listener 127.0.0.1 --port 9620 --db-uri sqlite:////tmp/vda.db > /tmp/vda_monitor.log 2>&1 &

Launch dn_agent
^^^^^^^^^^^^^^^

.. code-block:: none

   nohup vda_dn_agent --listener 127.0.0.1 --port 9720 --sock-path /tmp/dn.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}' > /tmp/vda_dn_agent.log 2>&1 &


Launch cn_agent
^^^^^^^^^^^^^^^

.. code-block:: none

   nohup vda_cn_agent --listener 127.0.0.1 --port 9820 --sock-path /tmp/cn.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}' > /tmp/vda_cn_agent.log 2>&1 &

Operate against the cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^

We have launched the dn_agent and cn_agent on the disk node and
controller node (both of them are localhost). But the cluster doesn't
record them to the database yet.

Run below command to add a disk node to the cluster:

.. code-block:: none

    vda_cli --addr-port 127.0.0.1:9520 dn create --dn-name localhost:9720 --dn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}'

Now we add a physical disk to the disk node. For testing purpose, we
create a malloc disk:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9720 --pd-name pd0 --pd-conf '{"type":"malloc","size":67108864}'

Add the controller node to the cluster:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 cn create --cn-name localhost:9820 --cn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}'

In a production environment, we may add lots of disk nodes and
contrtoller nodes. And we may have multiple physical disks in a single
disk node. In this tutorial we only have one disk ndoe and one
controller node, and only one malloc disk in the disk node.

Now we can create a disk array:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 da create --da-name da0 --cntlr-cnt 1 --da-size 33554432 --physical-size 33554432 --da-conf '{"stripe_count":1, "stripe_size_kb":64}'

The --da-name is the name of the disk array. We will use 'da0' to
refer the disk array. --cntlr-cnt means how many controller it will
have. We only have one controller node, we can not allocate more than
one controller for each disk array. --da-size is the size of the disk
array wil present to the user, --physical-size is the actual disk size
allocated from the disk node(s). Current VDA doesn't support extending
the physical size. So you should always set the --da-size
and --physical-size to the same value. --da-conf is the configuration
of the disk array. Current VDA only supports raid0. The parameter
stripe_count means how many legs the raid0 device has, stripe_size_kb
means the stripe size in KB of the raid0 device.

After creating the da0, we could gets its information:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 da get --da-name da0

The output should be something like below:

.. code-block:: none

   {
     "reply_info": {
       "req_id": "8e6ad92b082b4eceb3f13a522cbdd727",
       "reply_code": 0,
       "reply_msg": "success"
     },
     "da_msg": {
       "da_id": "e4f401c2890c4eb8b8f12bf8ea72b9d8",
       "da_name": "da0",
       "cntlr_cnt": 1,
       "da_size": 33554432,
       "da_conf": "{\"stripe_count\":1, \"stripe_size_kb\":64}",
       "da_details": "{\"lvs_conf\": {\"uuid\": \"ee843399-9888-4f65-a722-bbc708cf5984\", \"name\": \"vda-005-e4f401c2890c4eb8b8f12bf8ea72b9d8\", \"base_bdev\": \"vda-004-e4f401c2890c4eb8b8f12bf8ea72b9d8\", \"total_data_clusters\": 7, \"free_clusters\": 7, \"block_size\": 4096, \"cluster_size\": 4194304}}",
       "hash_code": 27012,
       "error": false
     },
     "grp_msg_list": [
       {
         "grp_id": "97a5f2d5b4984601833296255094e00b",
         "da_name": "da0",
         "grp_idx": 0,
         "grp_size": 33554432,
         "vd_msg_list": [
           {
             "vd_id": "5c50ec79a3e8453699f757b675132933",
             "da_name": "da0",
             "grp_idx": 0,
             "vd_idx": 0,
             "dn_name": "localhost:9720",
             "pd_name": "pd0",
             "vd_size": 33554432
           }
         ]
       }
     ],
     "cntlr_msg_list": [
       {
         "cntlr_id": "fab50f49958a4534828f6cf641c76ea3",
         "da_name": "da0",
         "cntlr_idx": 0,
         "cn_name": "localhost:9820",
         "primary": true,
         "error": false,
         "error_msg": ""
       }
     ]
   }

Export the disk array to localhost

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 exp create --da-name da0 --exp-name exp0 --initiator-nqn nqn.2016-06.io.spdk:host0

When we export the disk array, we give a --exp-name, the disk array
NQN will be generated from the da name and the exp
name. The --initiator-nqn is the NQN of the initiator, when the
initiator connect to this disk array, it should provide correct
initiator nqn, or the disk array will reject the connection.

Before let a host (initiator) connect to it, we need to know some
basic information, such as the disk array NQN, and detail parameters
about the NVMeoF protocal. We could use below command to get these
information:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 exp get --da-name da0 --exp-name exp0

The output should be something like below:

.. code-block:: none

   {
     "reply_info": {
       "req_id": "c6d5f10f2d3d4e9a9950313f909564a0",
       "reply_code": 0,
       "reply_msg": "success"
     },
     "exp_msg": {
       "exp_id": "5192e1f1cd154fbc92ebfb0743f5389e",
       "exp_name": "exp0",
       "exp_nqn": "nqn.2016-06.io.spdk:vda-exp-da0-exp0",
       "da_name": "da0",
       "initiator_nqn": "nqn.2016-06.io.spdk:host0",
       "snap_name": "",
       "es_msg_list": [
         {
           "es_id": "c99254393a4643d9b3da68cd78810d5c",
           "cntlr_idx": 0,
           "cn_name": "localhost:9820",
           "cn_listener_conf": "{\"trtype\":\"tcp\",\"traddr\":\"127.0.0.1\",\"adrfam\":\"ipv4\",\"trsvcid\":\"4430\"}",
           "error": false,
           "error_msg": ""
         }
       ]
     }
   }

We can find the disk array nqn from the "exp_nqn" field, which is
"nqn.2016-06.io.spdk:vda-exp-da0-exp0" in our example. We can find the
NVMeoF information from the cn_listener_conf field. In our example, we
know the protocal is tcp/ipv4, ip address is 127.0.0.1, port
is 4430. Then we could connect this disk array from the host.

Before connect to the disk array, we should make sure the nvme-tcp
kernel module is loaded and the nvme-cli is installed. Additinally, we
use the 'jq' command to extract the nvme information, so make sure it
is installed in the host too. Please run below commands:

.. code-block:: none

   sudo modprobe nvme-tcp
   sudo apt install -y nvme-cli
   sudo apt install jq

Connect to the disk array

.. code-block:: none

   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da0-exp0 -a 127.0.0.1 -s 4430 --hostnqn nqn.2016-06.io.spdk:host0

Then you can find the nvme device in /dev/nvme* . If you have multiple
nvme devices, you may use the "nvme list-subsys" command to find the
nqn of the disk array, then you can know which device is the disk
array we are connecting to. You may run below command:

.. code-block:: none

   sudo nvme list-subsys -o json | jq '.Subsystems[] | select(.NQN=="nqn.2016-06.io.spdk:vda-exp-da0-exp0")'

The output should be:

.. code-block:: none

   {
     "Name": "nvme-subsys0",
     "NQN": "nqn.2016-06.io.spdk:vda-exp-da0-exp0",
     "Paths": [
       {
         "Name": "nvme0",
         "Transport": "tcp",
         "Address": "traddr=127.0.0.1 trsvcid=4430",
         "State": "live"
       }
     ]
   }

From the output, we know the subsystem name is nvme0. In VDA, the
subsystem of the disk array will only have one name space. If the name
is nvme0, we can find the device in /dev/nvme0n1

We could try to access it:

.. code-block:: none

   sudo parted -s /dev/nvme0n1 print

The output should be:

.. code-block:: none

   Error: /dev/nvme0n1: unrecognised disk label
   Model: VDA_CONTROLLER (nvme)
   Disk /dev/nvme0n1: 33.6MB
   Sector size (logical/physical): 4096B/4096B
   Partition Table: unknown
   Disk Flags:

clean up all resoruces
^^^^^^^^^^^^^^^^^^^^^^

Disconnect the disk array from host

.. code-block:: none

   sudo nvme disconnect -n nqn.2016-06.io.spdk:vda-exp-da0-exp0

Delete the exporter

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 exp delete --da-name da0 --exp-name exp0

Delete the disk array

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 da delete --da-name da0

Delete the controller node

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 cn delete --cn-name localhost:9820

Delete the physical disk

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 pd delete --dn-name localhost:9720 --pd-name pd0

Delete the disk node

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 dn delete --dn-name localhost:9720

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

