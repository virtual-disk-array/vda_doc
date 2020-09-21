Multiple Serviers Deployment
============================

In this tutorial, we deploy the vda cluster components to multiple
servers. Below is the architecture:

.. image:: /images/multiple_servers_deployment.png

We use 10 servers, 3 for controller plane, 4 for data plane, 1 for the
client of the controller plane, 2 for the users of the data plane.

The 3 controller plane servers:

#. portal: Used for accept gRPC request. The portal is stateless, you
   could deploy multiple portals and put a load balancer at the front
   of them.
#. monitor: Used for syncup the cluster metadata to the disk nodes and
   controller nodes. You could deploy multiple monitors and let each
   monitor work on a subset of the disk nodes and controller
   nodes.
#. postgresql: It is the database, in this tutorial, we will use
   postgresql as the database. MySQL should work well too.

The 4 data plane servers:

#. cn0 and cn1: two controller nodes
#. dn0 and dn1: two data nodes

The vda_cli is the client of the controller plane. We send the gRPC
commands from this server, e.g. create a disk array, export a disk
array to a host and so on.

The host0 and host1 are the consumers of the disk arrays. After
vda_cli creates a disk array and exports to a host, the host could
connect to that disk array over the NVMeoF.

Similar as previous guides, in this tutorial, all of these servers are
ubuntu20.04. They could be deployed to other linux distrubution too.

Below is the server ip addresses and the server name:

192.168.0.10 postgresql

192.168.0.11 portal

192.168.0.12 monitor

192.168.0.13 cli

192.168.0.14 dn0

192.168.0.15 dn1

192.168.0.16 cn0

192.168.0.17 cn1

192.168.0.18 host0

192.168.0.19 host1

Install and configure postgresql
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Install postgresql from the default software repository, create user,
password and database.

.. code-block:: none

   sudo apt install -y postgresql
   sudo -u postgres psql
   create database vda_db;
   create user vda_user with encrypted password 'vda_password';
   grant all privileges on database vda_db to vda_user;
   exit

We should configure postgresql, let it allow connect from remote
clients. First, we should find the configuration file of
postgresql. We may run the ps command:

.. code-block:: none

   ps -f -C postgres

The output should be:

.. code-block:: none

   UID          PID    PPID  C STIME TTY          TIME CMD
   postgres     639       1  0 05:16 ?        00:00:00 /usr/lib/postgresql/12/bin/postgres -D /var/lib/postgresql/12/main -c config_file=/etc/postgresql/12/main/postgresql.conf
   postgres     660     639  0 05:16 ?        00:00:00 postgres: 12/main: checkpointer
   postgres     661     639  0 05:16 ?        00:00:00 postgres: 12/main: background writer
   postgres     662     639  0 05:16 ?        00:00:00 postgres: 12/main: walwriter
   postgres     663     639  0 05:16 ?        00:00:00 postgres: 12/main: autovacuum launcher
   postgres     664     639  0 05:16 ?        00:00:00 postgres: 12/main: stats collector
   postgres     665     639  0 05:16 ?        00:00:00 postgres: 12/main: logical replication launcher

Then we now the configuration file is
/etc/postgresql/12/main/postgresql.conf. Open this file, find
"listen_address = 'localhost'" and comment it, add "listen_addresses =
'*'"

.. code-block:: none

   listen_addresses = '*'
   #listen_addresses = 'localhost'         # what IP address(es) to listen on;

find the hba_file from the configuration file

.. code-block:: none

   cat /etc/postgresql/12/main/postgresql.conf | grep hba_file
   hba_file = '/etc/postgresql/12/main/pg_hba.conf'        # host-based authentication file

Open /etc/postgresql/12/main/pg_hba.conf, add below line:

.. code-block:: none

   host    all             all             192.168.0.0/24          md5

Then restart the postgresql:

.. code-block:: none

   sudo systemctl restart postgresql

Configure portal
^^^^^^^^^^^^^^^^
Install vda package:

.. code-block:: none

   sudo apt install -y python3-venv
   python3 -m venv vda_env
   source vda_env/bin/activate
   pip install vda

Install postgresql python client psycopg2

.. code-block:: none

   sudo apt install -y gcc
   sudo apt install -y python3-dev
   sudo apt install -y libpq-dev
   pip install wheel
   pip install psycopg2

Init the database, we only need to do it once when we create the
cluster

.. code-block:: none

   vda_db --action create --db-uri postgresql://vda_user:vda_password@192.168.0.10:5432/vda_db

Launch the portal process

.. code-block:: none

   nohup vda_portal --listener 192.168.0.11 --port 9520 --db-uri postgresql://vda_user:vda_password@192.168.0.10:5432/vda_db > /tmp/vda_portal.log 2>&1 &

Configure monitor
^^^^^^^^^^^^^^^^^
Install vda package:

Install vda package:

.. code-block:: none

   sudo apt install -y python3-venv
   python3 -m venv vda_env
   source vda_env/bin/activate
   pip install vda

Install postgresql python client psycopg2

.. code-block:: none

   sudo apt install -y gcc
   sudo apt install -y python3-dev
   sudo apt install -y libpq-dev
   pip install wheel
   pip install psycopg2

Launch the monitor process:

.. code-block:: none

   nohup vda_monitor --listener 192.168.0.12 --port 9620 --db-uri postgresql://vda_user:vda_password@192.168.0.10:5432/vda_db > /tmp/vda_monitor.log 2>&1 &

Configure dn0
^^^^^^^^^^^^^
Install spdk and init the spdk environment

.. code-block:: none

   cd ~
   git clone https://github.com/spdk/spdk
   cd spdk
   git submodule update --init
   sudo scripts/pkgdep.sh
   ./configure
   make
   sudo scripts/setup.sh

Launch the spdk application

.. code-block:: none

   nohup sudo ./build/bin/spdk_tgt --rpc-socket /tmp/dn.sock --wait-for-rpc > /tmp/dn.log 2>&1 &

Disable auto examine and change the sock file permission

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/dn.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/dn.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/dn.sock framework_wait_init
   sudo chmod 777 /tmp/dn.sock

Install vda package (we don't need to install postgresql python client
in data plane)

.. code-block:: none

   cd ~
   sudo apt install -y python3-venv
   python3 -m venv vda_env
   source vda_env/bin/activate
   pip install vda

Launch dn_agent

.. code-block:: none

   nohup vda_dn_agent --listener 192.168.0.14 --port 9720 --sock-path /tmp/dn.sock --listener-conf '{"trtype":"tcp","traddr":"192.168.0.14","adrfam":"ipv4","trsvcid":"4420"}' > /tmp/vda_dn_agent.log 2>&1 &

Configure dn1
^^^^^^^^^^^^^
Install spdk and init the spdk environment

.. code-block:: none

   cd ~
   git clone https://github.com/spdk/spdk
   cd spdk
   git submodule update --init
   sudo scripts/pkgdep.sh
   ./configure
   make
   sudo scripts/setup.sh

Launch the spdk application

.. code-block:: none

   nohup sudo ./build/bin/spdk_tgt --rpc-socket /tmp/dn.sock --wait-for-rpc > /tmp/dn.log 2>&1 &

Disable auto examine and change the sock file permission

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/dn.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/dn.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/dn.sock framework_wait_init
   sudo chmod 777 /tmp/dn.sock

Install vda package (we don't need to install postgresql python client
in data plane)

.. code-block:: none

   cd ~
   sudo apt install -y python3-venv
   python3 -m venv vda_env
   source vda_env/bin/activate
   pip install vda

Launch dn_agent

.. code-block:: none

   nohup vda_dn_agent --listener 192.168.0.15 --port 9720 --sock-path /tmp/dn.sock --listener-conf '{"trtype":"tcp","traddr":"192.168.0.15","adrfam":"ipv4","trsvcid":"4420"}' > /tmp/vda_dn_agent.log 2>&1 &

Configure cn0
^^^^^^^^^^^^^
Install spdk and init the spdk environment

.. code-block:: none

   cd ~
   git clone https://github.com/spdk/spdk
   cd spdk
   git submodule update --init
   sudo scripts/pkgdep.sh
   ./configure
   make
   sudo scripts/setup.sh

Launch the spdk application

.. code-block:: none

   nohup sudo ./build/bin/spdk_tgt --rpc-socket /tmp/cn.sock --wait-for-rpc > /tmp/cn.log 2>&1 &

Disable auto examine and change the sock file permission

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/cn.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/cn.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/cn.sock framework_wait_init
   sudo chmod 777 /tmp/cn.sock

Install vda package (we don't need to install postgresql python client
in data plane)

.. code-block:: none

   cd ~
   sudo apt install -y python3-venv
   python3 -m venv vda_env
   source vda_env/bin/activate
   pip install vda

Launch cn_agent

.. code-block:: none

   nohup vda_cn_agent --listener 192.168.0.16 --port 9820 --sock-path /tmp/cn.sock --listener-conf '{"trtype":"tcp","traddr":"192.168.0.16","adrfam":"ipv4","trsvcid":"4430"}' > /tmp/vda_cn_agent.log 2>&1 &

Configure cn1
^^^^^^^^^^^^^
Install spdk and init the spdk environment

.. code-block:: none

   cd ~
   git clone https://github.com/spdk/spdk
   cd spdk
   git submodule update --init
   sudo scripts/pkgdep.sh
   ./configure
   make
   sudo scripts/setup.sh

Launch the spdk application

.. code-block:: none

   nohup sudo ./build/bin/spdk_tgt --rpc-socket /tmp/cn.sock --wait-for-rpc > /tmp/cn.log 2>&1 &

Disable auto examine and change the sock file permission

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/cn.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/cn.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/cn.sock framework_wait_init
   sudo chmod 777 /tmp/cn.sock

Install vda package (we don't need to install postgresql python client
in data plane)

.. code-block:: none

   cd ~
   sudo apt install -y python3-venv
   python3 -m venv vda_env
   source vda_env/bin/activate
   pip install vda

Launch cn_agent

.. code-block:: none

   nohup vda_cn_agent --listener 192.168.0.17 --port 9820 --sock-path /tmp/cn.sock --listener-conf '{"trtype":"tcp","traddr":"192.168.0.17","adrfam":"ipv4","trsvcid":"4430"}' > /tmp/vda_cn_agent.log 2>& 1 &

Configure vda_cli
^^^^^^^^^^^^^^^^^

.. code-block:: none

   cd ~
   sudo apt install -y python3-venv
   python3 -m venv vda_env
   source vda_env/bin/activate
   pip install vda

Invoke VDA gRPC on vda_cli
^^^^^^^^^^^^^^^^^^^^^^^^^^
Add two dn nodes and create a malloc pd for each dn:

.. code-block:: none

   vda_cli --addr-port 192.168.0.11:9520 dn create --dn-name 192.168.0.14:9720 --dn-listener-conf '{"trtype":"tcp","traddr":"192.168.0.14","adrfam":"ipv4","trsvcid":"4420"}' --location 192.168.0.14:9720
   vda_cli --addr-port 192.168.0.11:9520 pd create --dn-name 192.168.0.14:9720 --pd-name pd0 --pd-conf '{"type":"malloc","size":134217728}'
   vda_cli --addr-port 192.168.0.11:9520 dn create --dn-name 192.168.0.15:9720 --dn-listener-conf '{"trtype":"tcp","traddr":"192.168.0.15","adrfam":"ipv4","trsvcid":"4420"}' --location 192.168.0.15:9720
   vda_cli --addr-port 192.168.0.11:9520 pd create --dn-name 192.168.0.15:9720 --pd-name pd0 --pd-conf '{"type":"malloc","size":134217728}'

Add two cn nodes:

.. code-block:: none

   vda_cli --addr-port 192.168.0.11:9520 cn create --cn-name 192.168.0.16:9820 --cn-listener-conf '{"trtype":"tcp","traddr":"192.168.0.16","adrfam":"ipv4","trsvcid":"4430"}' --location 192.168.0.16:9820
   vda_cli --addr-port 192.168.0.11:9520 cn create --cn-name 192.168.0.17:9820 --cn-listener-conf '{"trtype":"tcp","traddr":"192.168.0.17","adrfam":"ipv4","trsvcid":"4430"}' --location 192.168.0.16:9820

Create a disk array

.. code-block:: none

   vda_cli --addr-port 192.168.0.11:9520 da create --da-name da0 --cntlr-cnt 2 --da-size 33554432 --physical-size 33554432 --da-conf '{"stripe_count":2, "stripe_size_kb":64}'

Export da0 to host0

.. code-block:: none

   vda_cli --addr-port 192.168.0.11:9520 exp create --da-name da0 --exp-name exp0 --initiator-nqn nqn.2016-06.io.spdk:host0

Get the connection information

.. code-block:: none

   vda_cli --addr-port 192.168.0.11:9520 exp get --da-name da0 --exp-name exp0
   {
     "reply_info": {
       "req_id": "1901d2298e404ac8a27989c2f4da7a2e",
       "reply_code": 0,
       "reply_msg": "success"
     },
     "exp_msg": {
       "exp_id": "c4ab9583fd9842ba906fab3f5b536701",
       "exp_name": "exp0",
       "exp_nqn": "nqn.2016-06.io.spdk:vda-exp-da0-exp0",
       "da_name": "da0",
       "initiator_nqn": "nqn.2016-06.io.spdk:host0",
       "snap_name": "",
       "es_msg_list": [
         {
           "es_id": "0e31eecd1efc4fc2aa1054a5e1618c68",
           "cntlr_idx": 0,
           "cn_name": "192.168.0.16:9820",
           "cn_listener_conf": "{\"trtype\":\"tcp\",\"traddr\":\"192.168.0.16\",\"adrfam\":\"ipv4\",\"trsvcid\":\"4430\"}",
           "error": false,
           "error_msg": ""
         },
         {
           "es_id": "8ffd7919cb4140e49dc6baa9aaeb1aa0",
           "cntlr_idx": 1,
           "cn_name": "192.168.0.17:9820",
           "cn_listener_conf": "{\"trtype\":\"tcp\",\"traddr\":\"192.168.0.17\",\"adrfam\":\"ipv4\",\"trsvcid\":\"4430\"}",
           "error": false,
           "error_msg": ""
         }
       ]
     }
   }

Connect the da0 on host0
^^^^^^^^^^^^^^^^^^^^^^^^
Load nvme-tcp module, install nvme-cli and jq

.. code-block:: none

   sudo modprobe nvme-tcp
   sudo apt install -y nvme-cli
   sudo apt install -y jq

Connect to the two controller:

.. code-block:: none

   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da0-exp0 -a 192.168.0.16 -s 4430 --hostnqn nqn.2016-06.io.spdk:host0
   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da0-exp0 -a 192.168.0.17 -s 4430 --hostnqn nqn.2016-06.io.spdk:host0

Find the nvme device name from the NQN

.. code-block:: none

   sudo nvme list-subsys -o json | jq '.Subsystems[] | select(.NQN=="nqn.2016-06.io.spdk:vda-exp-da0-exp0")'
   {
     "Name": "nvme-subsys0",
     "NQN": "nqn.2016-06.io.spdk:vda-exp-da0-exp0",
     "Paths": [
       {
         "Name": "nvme0",
         "Transport": "tcp",
         "Address": "traddr=192.168.0.16 trsvcid=4430",
         "State": "live"
       },
       {
         "Name": "nvme1",
         "Transport": "tcp",
         "Address": "traddr=192.168.0.17 trsvcid=4430",
         "State": "live"
       }
     ]
   }

Access the device

.. code-block:: none

   sudo parted -s /dev/nvme0n1 print
   Error: /dev/nvme0n1: unrecognised disk label
   Model: VDA_CONTROLLER (nvme)
   Disk /dev/nvme0n1: 33.6MB
   Sector size (logical/physical): 4096B/4096B
   Partition Table: unknown
   Disk Flags:

Create another disk array on vda_cli
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Create da1

.. code-block:: none

   vda_cli --addr-port 192.168.0.11:9520 da create --da-name da1 --cntlr-cnt 2 --da-size 67108864 --physical-size 67108864 --da-conf '{"stripe_count":2, "stripe_size_kb":64}'

Export to host1:

.. code-block:: none

   vda_cli --addr-port 192.168.0.11:9520 exp create --da-name da1 --exp-name exp1 --initiator-nqn nqn.2016-06.io.spdk:host1

Get the connection information:

.. code-block:: none

   vda_cli --addr-port 192.168.0.11:9520 exp get --da-name da1 --exp-name exp1
   {
     "reply_info": {
       "req_id": "8b809cacfff241f9893933b0a112af43",
       "reply_code": 0,
       "reply_msg": "success"
     },
     "exp_msg": {
       "exp_id": "8aa4668dbd044dec939959dcaf8f902a",
       "exp_name": "exp1",
       "exp_nqn": "nqn.2016-06.io.spdk:vda-exp-da1-exp1",
       "da_name": "da1",
       "initiator_nqn": "nqn.2016-06.io.spdk:host1",
       "snap_name": "",
       "es_msg_list": [
         {
           "es_id": "faf04922b81a41c58c20e9228bfbcb59",
           "cntlr_idx": 0,
           "cn_name": "192.168.0.16:9820",
           "cn_listener_conf": "{\"trtype\":\"tcp\",\"traddr\":\"192.168.0.16\",\"adrfam\":\"ipv4\",\"trsvcid\":\"4430\"}",
           "error": false,
           "error_msg": ""
         },
         {
           "es_id": "572fd850517b4acfa50b8115e6c20781",
           "cntlr_idx": 1,
           "cn_name": "192.168.0.17:9820",
           "cn_listener_conf": "{\"trtype\":\"tcp\",\"traddr\":\"192.168.0.17\",\"adrfam\":\"ipv4\",\"trsvcid\":\"4430\"}",
           "error": false,
           "error_msg": ""
         }
       ]
     }
   }

Connect the da1 on host1
^^^^^^^^^^^^^^^^^^^^^^^^
Load nvme-tcp module, install nvme-cli and jq

.. code-block:: none

   sudo modprobe nvme-tcp
   sudo apt install -y nvme-cli
   sudo apt install -y jq

Connect to the two controller:

.. code-block:: none

   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da1-exp1 -a 192.168.0.16 -s 4430 --hostnqn nqn.2016-06.io.spdk:host1
   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da1-exp1 -a 192.168.0.17 -s 4430 --hostnqn nqn.2016-06.io.spdk:host1

Find the device name from NQN:

.. code-block:: none

   sudo nvme list-subsys -o json | jq '.Subsystems[] | select(.NQN=="nqn.2016-06.io.spdk:vda-exp-da1-exp1")'
   {
     "Name": "nvme-subsys0",
     "NQN": "nqn.2016-06.io.spdk:vda-exp-da1-exp1",
     "Paths": [
       {
         "Name": "nvme0",
         "Transport": "tcp",
         "Address": "traddr=192.168.0.16 trsvcid=4430",
         "State": "live"
       },
       {
         "Name": "nvme1",
         "Transport": "tcp",
         "Address": "traddr=192.168.0.17 trsvcid=4430",
         "State": "live"
       }
     ]
   }

Access the device:

.. code-block:: none

   sudo parted -s /dev/nvme0n1 print
   Error: /dev/nvme0n1: unrecognised disk label
   Model: VDA_CONTROLLER (nvme)
   Disk /dev/nvme0n1: 67.1MB
   Sector size (logical/physical): 4096B/4096B
   Partition Table: unknown
   Disk Flags:
