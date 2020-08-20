Getting started
===============

In this guide, we deploy all VDA components to a single service, and
describe the basic usage of it. There is no essential difference
between deploying all VDA components to a single server and deploying
them to different servers. The cluster data are stored in the
database, all other components are stateless.

Deployment
----------

requirements
^^^^^^^^^^^^
#. The vda is written by python3. You need python3 and venv to install
   the vda package
#. You need to install spdk on all the data node and controller node.
#. To use the virtual disk array, you need a host which support NVMeOF.

This getting started guide will show you how to install all the
requirements and all VDA components in a single ubunt 20.04 server.

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

   sudo HUGEMEM=5000 scripts/setup.sh

When run the spdk application, we should disable auto examine, so we
should use the --wait-for-rpc parameter, and invoke an API to disable
auto examine. We set the socket path to  /tmp/dn0.sock, later we will
pass this path to the dn_agent, then the dn_agent could communicate
with this spdk application.

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/dn0.sock --wait-for-rpc > /tmp/dn0.log 2>&1 &

We run below command to disable auto examine:

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/dn0.sock bdev_set_options -d

Then let the spdk application go to the normal mode:

.. code-block:: none

   sudo ./scripts/rpc.py -s /tmp/dn0.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/dn0.sock framework_wait_init

We do the simiar thing for the controller node spdk application:

.. code-block:: none

   sudo ./build/bin/spdk_tgt --rpc-socket /tmp/cn0.sock --wait-for-rpc > /tmp/cn0.log 2>&1 &
   sudo ./scripts/rpc.py -s /tmp/cn0.sock bdev_set_options -d
   sudo ./scripts/rpc.py -s /tmp/cn0.sock framework_start_init
   sudo ./scripts/rpc.py -s /tmp/cn0.sock framework_wait_init

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

