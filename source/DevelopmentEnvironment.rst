DevelopmentEnvironment
======================
Follow below steps to set up the development environment.

* Install golang, please refer the
  `golang official doc <https://golang.org/doc/install>`_

* Install below tools to the development environment:
  * git
  * make
  * protobuf compiler
  * gcc
  The install commands depend on the operation system. For ubuntu
  20.04 system, you can run below command to install them::

    sudo apt install -y git make protobuf-compiler gcc

* Install the build time golang module dependencies. Before this step,
  please make sure the ``go`` path has been set correctly, e.g.::

    export PATH=$PATH:/usr/local/go/bin

  Then run below commands::

    go get -u github.com/golang/protobuf/{proto,protoc-gen-go}
    go get github.com/golang/mock/mockgen

* Get the latest vda code then build it::

    git clone https://github.com/virtual-disk-array/vda
    cd vda
    make
