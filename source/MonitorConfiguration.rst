monitor configuration
=====================
The monitor is used to manager all the :ref:`DNs <dn-label>` and :ref:`CNs <cn-label>`.
Currently it has two kind of tasks:
* Send heartbeat request to all the ref:`DNs <dn-label>` and :ref:`CNs <cn-label>`.
  If any DN or CN can not response the heartbeat, it stores the DN/CN
  to etcd to indicate it is failed. And it also performs failover for
  the failed CN.
* For each failed DN/CN, the monitor gets the DN/CN data from etcd,
  and tries to syncup the data to the DN/CN. If it succeeds to syncup
  the DN/CN, it will remove the DN/CN from the failed list.

You can launch multiple monitors. Each monitor will work on a subset
of DNs and CNs.

command line parameters
-----------------------

--etcd-endpoints
  The etcd endpoint list, splited by comma. E.g. ``localhost:2379``,
  ``192.168.0.10:2379,192.168.0.11:2379,192.168.0.12:2379``. The default
  value is localhost:2379.

--dn-heartbeat-interval
  The interval of sending heartbeat request to each DN. Its unit is
  second. If set it to 0, the dn heartbeat will be disabled. The default
  value is 5.

--dn-heartbeat-concurrency
  How many goroutines will be launched for sending DN heartbeat
  request. If you have many DNs, you should set it to a larger value (or
  launch more monitors). If set it to 0, the dn heartbeat will be
  disabled. The default value is 100.


--dn-syncup-interval
  The interval of sending syncup request to the failed DNs. Its unit is
  second. If set it to 0, the dn syncup will be disabled. The default
  value is 5.

--dn-syncup-concurrency
  How many goroutines will be launched for sending the syncup
  request. If set it to 0, the dn syncup will be disabled. The default
  value is 100.

--cn-heartbeat-interval
  The interval of sending heartbeat request to each CN. Its unit is
  second. If set it to 0, the cn heartbeat will be disabled. The default
  value is 5.

--cn-heartbeat-concurrency
  How many goroutines will be launched for sending CN heartbeat
  request. If you have many CNs, you should set it to a larger value (or
  launch more monitors). If set it to 0, the cn heartbeat will be
  disabled. The default value is 100.

--cn-syncup-interval
  The interval of sending syncup request to the failed CNs. Its unit is
  second. If set it to 0, the cn syncup will be disabled. The default
  value is 5.

--cn-syncup-concurrency
  How many goroutines will be launched for sending the syncup
  request. If set it to 0, the cn syncup will be disabled. The default
  value is 100.

examples
--------

* Connect to the etcd 192.168.0.10:2379, and use default values for all
  other parameters::

    vda_monitor --etcd-endpoints 192.168.0.10:2379

* Connet to the etcd cluster 192.168.0.10:2379, 192.168.0.11:2379 and
  192.168.0.12:2379, Set dn heartbeat interval to 10 second::

    vda_monitor --etcd-endpoints 192.168.0.10:2379,192.168.0.11:2379,192.168.0.12:2379 --dn-heartbeat-interval 10
    
