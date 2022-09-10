DevelopmentEnvironment
======================
Follow below steps to set up the development environment.

* Install the latest version of below tools to your development system:

  * git
  * make
  * curl
  * unzip
  * gcc
  * jq
  * zipk
  * pkg-config

.. note::

   If you are using ubuntu system, the ``sudo apt install pkg-config``
   command may install a pretty old pkg-config which doesn't work for
   the vda dataplane application. Please try below command to install a
   new pkg-config: ``sudo apt install pkgconf``

* Install golang

  The recommended golang vesion is ``1.18.3``. An older version may not
  work. You may follow below steps, they are copied and modified from
  the `golang official doc <https://golang.org/doc/install>`_::

    curl -L -O https://go.dev/dl/go1.18.3.linux-amd64.tar.gz
    rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.18.3.linux-amd64.tar.gz

  Then add ``/usr/local/go/bin`` to the ``PATH`` environment variable, you
  may add below lines to your $HOME/.profile::

    export PATH="$PATH:/usr/local/go/bin"

  Run below command to verify golang is installed::

    go version

* Install protoc

  The `gRPC official doc <https://grpc.io/docs/protoc-installation/>`_
  provide multiple ways to install protoc. Below is an OS
  independently way::

    curl -L -O https://github.com/protocolbuffers/protobuf/releases/download/v21.2/protoc-21.2-linux-x86_64.zip
    unzip protoc-21.2-linux-x86_64.zip -d $HOME/.local

  Export ``$HOME/.local/bin`` to the ``PATH`` environment variable::

    export PATH="$PATH:$HOME/.local/bin"

* Check out VDA source code and install golang build dependencies

  Check out VDA source code::

    git clone https://github.com/virtual-disk-array/vda

  Go to the ``vda`` directory::

    cd vda

  Install the dataplane dependencies::

    ./scripts/dataplane/dataplane_prepare.sh

  The above script will invoke the ``pkgdep.sh`` of the
  SPDK. If you get any problem, you may try to follow the
  `SPDK Getting Started <https://spdk.io/doc/getting_started.html>`_ ,
  verify whether you can build the official SPDK without any probelm.

  Run ``go install`` to install the controlplane build dependencies::

    go install google.golang.org/protobuf/cmd/protoc-gen-go
    go install google.golang.org/grpc/cmd/protoc-gen-go-grpc
    go install github.com/golang/mock/mockgen

  Finally you can run the ``make`` command to compile VDA::

    make

* Instal ``gh`` and zip (optinal).

  We need github cli ``gh`` to make a release. Find the package from
  the `gh release page <https://github.com/cli/cli/releases/latest>`_
