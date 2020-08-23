Multiple Components In One Server
=================================

In a production envirnment, to get high availability, we should have
at least two instances for each control plane components (portal and
monitor), and many instances for each data plane components (cn_agent
and dn_agent). In this guide, we deploy two instances for each
components. As a demo, we deploy all of them to a single server. In
the next guide, we will deploy them to different servers.

.. image:: /images/multiple_components_in_one_server.png

Each cn_agent and dn_agent should have its own spdk application. We
plan to deploy 2 cn_agent and 2 dn_agent, so we will launch 4 spdk
applications. All of the should listen on different port. We only have
one database instance. Here we will use Postgresql as an example. It
should be MySQL too. The HA of database is out of the scope of this
document. You should follow the general HA solution(s) of the database
you use.

Here we use ubuntu20.04 as an example system. But they should be
deployed to most of the linux distributions. The server should have at
least 16G memory, and we will allocate 8G huge page for spdk
applications.

Install
-------

make sure system is up to date
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: none

   sudo apt update -y && sudo apt upgrade -y && sudo reboot

install spdk
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

run spdk
^^^^^^^^
We will have 2 controller nodes and two disk nodes. So we will launch
4 spdk applications. And we should disable auto examine on all of
them. Befor launch any spdk application, we should initialize the
spdk environment. Please run below commands under the spdk directory:

.. code-block:: none

   sudo HUGEMEM=8192 scripts/setup.sh

Please note, if you don't hvae 8192M huge page, please set
HUGEMEM=5000 at least. The 4096M may not work for 4 spdk applications.

Launch the dn_spdk_app_0

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/dn0.sock --wait-for-rpc > /tmp/dn0.log 2>&1 &
   sudo ./scripts/rpc.py -s /tmp/dn0.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/dn0.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/dn0.sock framework_wait_init
   sudo chmod 777 /tmp/dn0.sock

Launch the dn_spdk_app_1

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/dn1.sock --wait-for-rpc > /tmp/dn1.log 2>&1 &
   sudo ./scripts/rpc.py -s /tmp/dn1.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/dn1.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/dn1.sock framework_wait_init
   sudo chmod 777 /tmp/dn1.sock

Launch the cn_spdk_app_0

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/cn0.sock --wait-for-rpc > /tmp/cn0.log 2>&1 &
   sudo ./scripts/rpc.py -s /tmp/cn0.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/cn0.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/cn0.sock framework_wait_init
   sudo chmod 777 /tmp/cn0.sock

launch the cn_spdk_app_1

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/cn1.sock --wait-for-rpc > /tmp/cn1.log 2>&1 &
   sudo ./scripts/rpc.py -s /tmp/cn1.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/cn1.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/cn1.sock framework_wait_init
   sudo chmod 777 /tmp/cn1.sock

install vda
^^^^^^^^^^^
install venv, create a python virtual environment, install vda in this
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

install postgresql
^^^^^^^^^^^^^^^^^^
install the database, create db, user, and grant all permission to the
user:

.. code-block:: none

   sudo apt install -y postgresql
   sudo -u postgres psql
   create database vda_db;
   create user vda_user with encrypted password 'vda_password';
   grant all privileges on database vda_db to vda_user;
   exit

To let vda communicate with postgresql, you need to install the db
driver psycopg2. The psycopg2 is a python wrapper for libpq. Before
install psycopg2, you need to install below packages on your system:

.. code-block:: none

   sudo apt install -y gcc
   sudo apt install python3-dev
   sudo apt install libpq-dev

Then make sure the current terminal is in the vda_env virtual
enviornment, and run:

.. code-block:: none

   pip install psycopg2

init database
^^^^^^^^^^^^^

.. code-block:: none

   vda_db --action create --db-uri postgresql://vda_user:vda_password@localhost:5432/vda_db

launch two portals
^^^^^^^^^^^^^^^^^^

.. code-block:: none

   vda_portal --listener 127.0.0.1 --port 9520 --db-uri postgresql://vda_user:vda_password@localhost:5432/vda_db > /tmp/vda_portal_0.log 2>&1 &
   vda_portal --listener 127.0.0.1 --port 9521 --db-uri postgresql://vda_user:vda_password@localhost:5432/vda_db > /tmp/vda_portal_1.log 2>&1 &

launch two monitors
^^^^^^^^^^^^^^^^^^^

.. code-block:: none

   vda_monitor --listener 127.0.0.1 --port 9620 --db-uri postgresql://vda_user:vda_password@localhost:5432/vda_db > /tmp/vda_monitor_0.log 2>&1 &
   vda_monitor --listener 127.0.0.1 --port 9621 --db-uri postgresql://vda_user:vda_password@localhost:5432/vda_db > /tmp/vda_monitor_1.log 2>&1 &

launch two dn_agents
^^^^^^^^^^^^^^^^^^^^

.. code-block:: none

   vda_dn_agent --listener 127.0.0.1 --port 9720 --sock-path /tmp/dn0.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}' > /tmp/vda_dn_agent_0.log 2>&1 &
   vda_dn_agent --listener 127.0.0.1 --port 9720 --sock-path /tmp/dn1.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4421"}' > /tmp/vda_dn_agent_1.log 2>&1 &

launch two cn_agents
^^^^^^^^^^^^^^^^^^^^

.. code-block:: none

   vda_cn_agent --listener 127.0.0.1 --port 9820 --sock-path /tmp/cn0.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}' > /tmp/vda_cn_agent_0.log 2>&1 &
   vda_cn_agent --listener 127.0.0.1 --port 9821 --sock-path /tmp/cn1.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4431"}' > /tmp/vda_cn_agent_0.log 2>&1 &

create several disk arrays
^^^^^^^^^^^^^^^^^^^^^^^^^^
Before create any disk array, we should add the two controller nodes
and the two disk ndoes to the system, and create physical disks on
each disk node.

create two disk nodes:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 dn create --dn-name localhost:9720 --dn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}'
   vda_cli --addr-port 127.0.0.1:9520 dn create --dn-name localhost:9721 --dn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4421"}'

prepare 4 files, will use them as physical disks:

.. code-block:: none

   dd if=/dev/zero of=/tmp/a.img bs=1M count=1 seek=1023
   dd if=/dev/zero of=/tmp/b.img bs=1M count=1 seek=1023
   dd if=/dev/zero of=/tmp/c.img bs=1M count=1 seek=1023
   dd if=/dev/zero of=/tmp/d.img bs=1M count=1 seek=1023

Add the four physical disks to the two disk nodes, each node has two
disks:

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9720 --pd-name pd0 --pd-conf '{"type":"aio","filename":"/tmp/a.img"}'
   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9720 --pd-name pd1 --pd-conf '{"type":"aio","filename":"/tmp/b.img"}'
   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9721 --pd-name pd0 --pd-conf '{"type":"aio","filename":"/tmp/c.img"}'
   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9721 --pd-name pd1 --pd-conf '{"type":"aio","filename":"/tmp/d.img"}'

Create two controller nodes

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 cn create --cn-name localhost:9820 --cn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}'
   vda_cli --addr-port 127.0.0.1:9520 cn create --cn-name localhost:9821 --cn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4431"}'

Create a disk array

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 da create --da-name da0 --cntlr-cnt 2 --da-size 33554432 --physical-size 33554432 --da-conf '{"stripe_count":2, "stripe_size_kb":64}'

Export the disk array da0 to localhost

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 exp create --da-name da0 --exp-name exp0 --initiator-nqn nqn.2016-06.io.spdk:host0

Before connect the disk array to the current host, make sure the
nvme-tcp module is loaded and the nvme-cli is installed:

.. code-block:: none

   sudo modprobe nvme-tcp
   sudo apt install -y nvme-cli

Discover and connect the disk array, we have two controllers, we
should discover and connect the two controllers separately.

Discover and connect to the first controller:

.. code-block:: none

   sudo nvme discover -t tcp -a 127.0.0.1 -s 4430 --hostnqn nqn.2016-06.io.spdk:host0
   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da0-exp0 -a 127.0.0.1 -s 4430 --hostnqn nqn.2016-06.io.spdk:host0

Discover and connect to the second controller:

.. code-block:: none

   sudo nvme discover -t tcp -a 127.0.0.1 -s 4431 --hostnqn nqn.2016-06.io.spdk:host0
   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da0-exp0 -a 127.0.0.1 -s 4431 --hostnqn nqn.2016-06.io.spdk:host0
   

If you have no other nvme device, you can find nvme0 and nvme1 under
the /dev directory. And you can find a /dev/nvme0n1, there is no
/dev/nvme1n1. The /dev/nvme0n1 is a multipath device. If one
controller doesn't work, the kernel driver will failover to another
one automatically.

clean up all resoruces
^^^^^^^^^^^^^^^^^^^^^^

Disconnect the disk array

.. code-block:: none

   sudo nvme disconnect -n nqn.2016-06.io.spdk:vda-exp-da0-exp0

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

Drop the database

.. code-block:: none

   vda_db --action drop --db-uri postgresql://vda_user:vda_password@localhost:5432/vda_db

Kill all processes

.. code-block:: none

   killall vda_portal
   killall vda_monitor
   killall vda_dn_agent
   killall vda_cn_agent
   sudo killall reactor_0
