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

As a demo, We deploay both the kubernetes cluster and the vda
compoents to the same server.

Deploy VDA
----------
The steps are similar as the :ref:`Minimal Deployment <minimal-deployment-label>`.

* Create a work directory::

    mkdir -p /tmp/vda_data

* Install etcd and launch::

    curl -L -O https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
    tar xvf etcd-v3.5.0-linux-amd64.tar.gz
    etcd-v3.5.0-linux-amd64/etcd --listen-client-urls http://localhost:2389 \
    --advertise-client-urls http://localhost:2389 \
    --listen-peer-urls http://localhost:2390 \
    --name etcd0 --data-dir /tmp/vda_data/etcd0.data \
    > /tmp/vda_data/etcd0.log 2>&1 &


* Download vda and got to the vda directory::

    curl -L -O https://github.com/virtual-disk-array/vda/releases/download/v0.2.0/vda_linux_amd64_v0.2.0.tar.gz
    tar xvf vda_linux_amd64_v0.2.0.tar.gz
    cd vda_linux_amd64_v0.2.0

* Prepare SPDK environment::

    sudo ./spdk/scripts/setup.sh

* Launch DN dataplane::

    sudo ./vda_dataplane --config ./dataplane_config.json \
    --rpc-socket /tmp/vda_data/dn.sock > /tmp/vda_data/dn.log 2>&1 &

* Change the ower of dn.sock::

    sudo chown $(id -u):$(id -g) /tmp/vda_data/dn.sock

* Launch DN controplane::

    ./vda_dn_agent --network tcp --address '127.0.0.1:9720' \
    --sock-path /tmp/vda_data/dn.sock --sock-timeout 10 \
    --lis-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4420"}' \
    --tr-conf '{"trtype":"TCP"}' \
    > /tmp/vda_data/dn_agent.log 2>&1 &

* Launch CN dataplane::

    ./vda_cn_agent --network tcp --address '127.0.0.1:9820' \
    --sock-path /tmp/vda_data/cn.sock --sock-timeout 10 \
    --lis-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}' \
    --tr-conf '{"trtype":"TCP"}' \
    > /tmp/vda_data/cn_agent.log 2>&1 &

* Change the ower of cn.sock,::

    sudo chown $(id -u):$(id -g) /tmp/vda_data/cn.sock

* Launch CN controlplane::

    ./vda_cn_agent --network tcp --address '127.0.0.1:9820' \
    --sock-path /tmp/vda_data/cn.sock --sock-timeout 10 \
    --lis-conf '{"trtype":"tcp","traddr":"127.0.0.1","adrfam":"ipv4","trsvcid":"4430"}' \
    --tr-conf '{"trtype":"TCP"}' \
    > /tmp/vda_data/cn_agent.log 2>&1 &

* Launch portal::

    ./vda_portal --portal-address '127.0.0.1:9520' --portal-network tcp \
    --etcd-endpoints localhost:2389 \
    > /tmp/vda_data/portal.log 2>&1 &

* Launch monitor::

    ./vda_monitor --etcd-endpoints localhost:2389 \
    > /tmp/vda_data/monitor.log 2>&1 &

* Create DN::

    ./vda_cli dn create --sock-addr localhost:9720 \
    --tr-type tcp --tr-addr 127.0.0.1 --adr-fam ipv4 --tr-svc-id 4420

* Create PD::

    ./vda_cli pd create --sock-addr localhost:9720 --pd-name pd0 \
    --bdev-type-key malloc --bdev-type-value 256

* Create CN::

    ./vda_cli cn create --sock-addr localhost:9820 \
    --tr-type tcp --tr-addr 127.0.0.1 --adr-fam ipv4 --tr-svc-id 4430

That's all. We won't create DAs and EXPs manually here. The kubernetes
will create them on demand.

Deploy Kubernetes
-----------------
As a demo, we deploy a minimal kuberenetes cluster via minikube.

Install minikube
^^^^^^^^^^^^^^^^
According to the `minikube installation guide <https://minikube.sigs.k8s.io/docs/start/>`_
You should run below commands on a linux server::

  curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
  sudo install minikube-linux-amd64 /usr/local/bin/minikube


Prepare the dependencies of none driver
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The minikube support `multiple drivers <https://minikube.sigs.k8s.io/docs/drivers/>`_
to launch kuberenetes. The vda csi plugin in kuberentes need to access
the /dev and /sys of the host. Here we use the minikube none driver
because the other minikube drivers require additional steps to
transport the /dev and /sys to the vda csi plugin.

Please refer the `official doc <https://minikube.sigs.k8s.io/docs/drivers/none/#requirements>`_
about the dependencies of none drivers. The installation steps are OS
dependent. Below steps are tested on ubuntu22.04 and rocky linux 8,
and they should work on other linux systems too.

* Follow the `docker installation guide <https://docs.docker.com/engine/install/#server>`_
  to install docker engine. Please choose the ``Platform`` according
  to your OS.

* Manage docker as a non-root user according to the `post installation guide <https://docs.docker.com/engine/install/linux-postinstall/>`_.

* Disable selinux. You may run the ``getenforce`` command from command
  line. If the output is ``enforcing``, you should disable selinux. If your
  system doesn't have such a command, the selinux probably is not
  enabled, you may ignore this step. To disable selinux, please edit
  the ``/etc/selinux/config`` file, change ``SELINUX=enforcing`` to
  ``SELINUX=permissive``.

* Install cri-tools. Open the `cri-tools release page <https://github.com/cri-o/cri-o/releases/latest>`_,
  download the released package according to your OS, then uncompress
  the package and run the install script, e.g.::

    curl -L -O https://github.com/cri-o/cri-o/releases/download/v1.25.0/cri-o.amd64.v1.25.0.tar.gz
    tar xvf cri-o.amd64.v1.25.0.tar.gz
    cd cri-o
    sudo ./install

* The crictl will be install to ``/usr/local/bin``. The minikube will
  invoke it over the ``sudo`` command. Please run ``sudo visudo`` and
  make sure the ``/usr/local/bin`` is in the ``secure_path``. If no,
  please add it to the ``secure_path``, e.g.::

    Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin

* Install cri-dockerd. Open the `cri-dockerd release page <https://github.com/Mirantis/cri-dockerd/releases/latest>`_,
  download the package according to your OS and install it. E.g.::

    # rocky linux 8:
    curl -L -O https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.5/cri-dockerd-0.2.5-3.el8.x86_64.rpm
    sudo rpm -i cri-dockerd-0.2.5-3.el8.x86_64.rpm
    # ubuntu22.04:
    curl -L -O https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.5/cri-dockerd_0.2.5.3-0.ubuntu-jammy_amd64.deb
    sudo dpkg -i cri-dockerd_0.2.5.3-0.ubuntu-jammy_amd64.deb

*  Install conntrack. The package name might be different in different
   OS::

     sudo dnf install -y conntrack-tools # rocky linux 8
     sudo apt-get install -y conntrack # ubuntu22.04

Launch kubernetes cluster
^^^^^^^^^^^^^^^^^^^^^^^^^
Run below command::

  minikube start --driver=none

Run below command to check all kubernentes components are available::

  minikube kubectl -- get pods -A

The result should be::

  NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
  kube-system   coredns-6d4b75cb6d-s65dv              1/1     Running   0          5s
  kube-system   etcd-ubuntu2204b                      1/1     Running   0          18s
  kube-system   kube-apiserver-ubuntu2204b            1/1     Running   0          20s
  kube-system   kube-controller-manager-ubuntu2204b   1/1     Running   0          20s
  kube-system   kube-proxy-mvw4k                      1/1     Running   0          6s
  kube-system   kube-scheduler-ubuntu2204b            1/1     Running   0          18s
  kube-system   storage-provisioner                   1/1     Running   0          17s

You may wait several minutes or a longer time for all components are
ready.
 
Prepare the vda csi plugin dependencies
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
On each `kubernetes node <https://kubernetes.io/docs/concepts/architecture/nodes/>`_
we should install the nvme-cli and load nvme-tcp kernel module::

  sudo dnf install -y nvme-cli # rocky linux 8
  sudo apt install -y nvme-cli # ubuntu22.04
  sudo modprobe nvme-tcp

Create sidecars
^^^^^^^^^^^^^^^
Download the example configuration files::

  curl -L -O https://raw.githubusercontent.com/virtual-disk-array/vda/master/scripts/integtest/controller-rbac.yaml
  curl -L -O https://raw.githubusercontent.com/virtual-disk-array/vda/master/scripts/integtest/controller.yaml
  curl -L -O https://raw.githubusercontent.com/virtual-disk-array/vda/master/scripts/integtest/node-rbac.yaml
  curl -L -O https://raw.githubusercontent.com/virtual-disk-array/vda/master/scripts/integtest/node.yaml
  curl -L -O https://raw.githubusercontent.com/virtual-disk-array/vda/master/scripts/integtest/storageclass.yaml

Apply them to kubernetes::

  minikube kubectl -- apply -f controller-rbac.yaml
  minikube kubectl -- apply -f controller.yaml
  minikube kubectl -- apply -f node-rbac.yaml
  minikube kubectl -- apply -f node.yaml
  minikube kubectl -- apply -f storageclass.yaml

Get the status of the controller and node::

  minikube kubectl -- get pods

Make sure the ``READY`` of controller and node become ``3/3`` and ``2/2``::

  NAME                  READY   STATUS    RESTARTS   AGE
  vdacsi-controller-0   3/3     Running   0          17s
  vdacsi-node-rng9x     2/2     Running   0          17s

Create testing pod
------------------
Download the sample PVC(PersistentVolumeClaim) and Pod file ::

  curl -L -O https://raw.githubusercontent.com/virtual-disk-array/vda/master/scripts/integtest/testpvc.yaml
  curl -L -O https://raw.githubusercontent.com/virtual-disk-array/vda/master/scripts/integtest/testpod.yaml

Apply the PVC file ::

  minikube kubectl -- apply -f testpvc.yaml

Apply the Pod file ::

  minikube kubectl -- apply -f testpod.yaml

Wait for a while, run below command to get the status of the testpod::

  minikube kubectl -- get pods vdacsi-test

You would get similar output as below::

  NAME          READY   STATUS    RESTARTS   AGE
  vdacsi-test   1/1     Running   0          55s

Cleanup
-------

Cleanup the kubernetes cluster
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
* Delete test pod and test PVC::

    minikube kubectl -- delete pod vdacsi-test
    minikube kubectl -- delete pvc vdacsi-pvc

* Delete the kubernentes cluster::

    minikube stop
    minikube delete

Cleanup the VDA cluster
^^^^^^^^^^^^^^^^^^^^^^^
* Kill all the processes::

    killall vda_portal
    killall vda_monitor
    killall vda_dn_agent
    killall vda_cn_agent
    killall etcd
    cd vda_linux_amd64_v0.2.0
    ./spdk/scripts/rpc.py -s /tmp/vda_data/dn.sock spdk_kill_instance SIGTERM
    ./spdk/scripts/rpc.py -s /tmp/vda_data/cn.sock spdk_kill_instance SIGTERM

* Delete the work directory::

    rm -rf /tmp/vda_data
