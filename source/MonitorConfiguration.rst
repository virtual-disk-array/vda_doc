monitor configuration
=====================

command line parameters
-----------------------

\--listener
^^^^^^^^^^^
The grpc listen address. You may use 0.0.0.0 to let it listen on all
address. Or use 127.0.0.1 to listen on the local address. Or a
specific ip address to let it only listen on that address,
e.g. 182.168.0.10. The default value is 127.0.0.1.

\--port
^^^^^^^
The grpc listen port. The default value is 9620.

\--max-workers
^^^^^^^^^^^^^^
It indicate how many concurrent workers the grpc server will have. The
default value is 10.

\--db-uri, \--db-uri-file, \--db-uri-stdin
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The above 3 parameters are exclusive with each other. You may choose
one of them to set the database uri. The database uri will be pass to
the sqlalchemy. Please refer the
`sqlalchemy doc <https://docs.sqlalchemy.org/en/13/core/engines.html>`_
for its format.

* \--db-uri means set the db uri from the command line directly.
* \--db-uri-file means read the db uri from a local file.
* \--db-uri-stdin means read the db uri from stdin. Then the user should
  write the db uri to the stdin of vda_portal.

Below are several examples of the db uri.

sqlite:

.. code-block:: none

   sqlite:////tmp/vda.db

postgresql:

.. code-block:: none

   postgresql://vda_user:vda_password@localhost:5432/vda_db

\--db-kwargs
^^^^^^^^^^^^
It is a json string which will be passed to the
`create_engine function of sqlalchemy <https://docs.sqlalchemy.org/en/13/core/engines.html#sqlalchemy.create_engine>`_
directly.

\--total, \--current
^^^^^^^^^^^^^^^^^^^^
The parameter 'total' means how many monitors totally, 'current' means
the index of this monitor.
