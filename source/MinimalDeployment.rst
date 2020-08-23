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
distributions. The server should have at least 8G memory, and we will
allocate 4G huge page for spdk applications.

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
In this tutorial, we plan to run one controller node and one disk
node. Each node should have its own spdk application. So we will
launch two spdk applications.
Before launch the spdk application, we need to initialize the spdk
environment, we could run below script in the spdk directory.

.. code-block:: none

   sudo HUGEMEM=4096 scripts/setup.sh

When run the spdk application, we should disable auto examine, so we
should use the --wait-for-rpc parameter, and invoke an API to disable
auto examine. We set the socket path to  /tmp/dn0.sock, later we will
pass this path to the dn_agent, then the dn_agent could communicate
with this spdk application.

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/dn.sock --wait-for-rpc > /tmp/dn.log 2>&1 &

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

We do the similar things for the controller node spdk application:

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/cn.sock --wait-for-rpc > /tmp/cn.log 2>&1 &
   sudo ./scripts/rpc.py -s /tmp/cn.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/cn.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/cn.sock framework_wait_init
   sudo chmod 777 /tmp/cn.sock

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

init database
^^^^^^^^^^^^^

.. code-block:: none

   vda_db --action create --db-uri sqlite:////tmp/vda.db

launch portal
^^^^^^^^^^^^^

.. code-block:: none

   vda_portal --listener 127.0.0.1 --port 9520 --db-uri sqlite:////tmp/vda.db > /tmp/vda_portal.log 2>&1 &


launch monitor
^^^^^^^^^^^^^^

.. code-block:: none

   vda_monitor --listener 127.0.0.1 --port 9620 --db-uri sqlite:////tmp/vda.db > /tmp/vda_monitor.log 2>&1 &

launch dn_agent
^^^^^^^^^^^^^^^

.. code-block:: none

   vda_dn_agent --listener 127.0.0.1 --port 9720 --sock-path /tmp/dn.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}' > /tmp/vda_dn_agent.log 2>&1 &


launch cn_agent
^^^^^^^^^^^^^^^

.. code-block:: none

   vda_cn_agent --listener 127.0.0.1 --port 9820 --sock-path /tmp/cn.sock --listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}' > /tmp/vda_cn_agent.log 2>&1 &

create a disk array
^^^^^^^^^^^^^^^^^^^
Before create any virtual disk array, we should add the controller
node and the disk node to the system, and create a physical disk on
the disk node. Then we can create the virtual disk arrary.

create a disk node

.. code-block:: none

    vda_cli --addr-port 127.0.0.1:9520 dn create --dn-name localhost:9720 --dn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}'

create a physical disk

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 pd create --dn-name localhost:9720 --pd-name pd0 --pd-conf '{"type":"malloc","size":67108864}'

create a controller node

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 cn create --cn-name localhost:9820 --cn-listener-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}'

create a disk array

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 da create --da-name da0 --cntlr-cnt 1 --da-size 33554432 --physical-size 33554432 --da-conf '{"stripe_count":1, "stripe_size_kb":64}'

export the disk array to localhost

.. code-block:: none

   vda_cli --addr-port 127.0.0.1:9520 exp create --da-name da0 --exp-name exp0 --initiator-nqn nqn.2016-06.io.spdk:host0

We connect the disk arrary from the current host. Before connect it,
make sure the nvme-tcp module is loaded and the nvme-cli is installed

.. code-block:: none

   sudo modprobe nvme-tcp
   sudo apt install -y nvme-cli

Discover the disk array

.. code-block:: none

   sudo nvme discover -t tcp -a 127.0.0.1 -s 4430 --hostnqn nqn.2016-06.io.spdk:host0

Connect to the disk array

.. code-block:: none

   sudo nvme connect -t tcp -n nqn.2016-06.io.spdk:vda-exp-da0-exp0 -a 127.0.0.1 -s 4430 --hostnqn nqn.2016-06.io.spdk:host0

Then you can find the nvme device in /dev/nvme*

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

Drop the database

.. code-block:: none

   vda_db --action drop --db-uri sqlite:////tmp/vda.db

Kill all processes

.. code-block:: none

   killall vda_portal
   killall vda_monitor
   killall vda_dn_agent
   killall vda_cn_agent
   sudo killall reactor_0
