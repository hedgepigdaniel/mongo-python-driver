Testing PyMongo with mod_wsgi
=============================

These tests verify that PyMongo works with Apache and mod_wsgi. They are
primarily intended to prevent regression of
`PYTHON-353 <https://jira.mongodb.org/browse/PYTHON-353>`_, a connection leak
when PyMongo 2.2 was used with Python 2.6 and mod_wsgi 2.8. However, the test
may also catch concurrency bugs, or incompatibilities between PyMongo's C
extensions and the way mod_wsgi manages Python sub interpreters. It is
generally useful to test PyMongo in the unconventional environment that
mod_wsgi creates.

Test Matrix
-----------

PyMongo should be tested with several versions of mod_wsgi and a selection
of Python versions. Each combination of mod_wsgi and Python version should
be tested with a standalone and a replica set. ``mod_wsgi_test.wsgi``
detects if the deployment is a replica set and connects to the whole set.

Setup
-----

Compile Python
..............

We need a Python interpreter built as a shared library. Download the
source tarball for each Python version tested, untar it, and run::

    ./configure --prefix=/some/directory --enable-shared
    make
    make install

This results in an executable named "python" and a shared
library named something like "libpython2.7.so.1.0".
It may be necessary to add /some/directory/lib to your system's
LD_LIBRARY_PATH, or to make a symlink from your system's default library
directory to the shared library. For example, on Ubuntu::

    ln -s /some/directory/lib/libpython2.7.so.1.0 /usr/lib64/

Compile mod_wsgi
................

Compile mod_wsgi for each combination for Python and mod_wsgi version in the
test matrix. For example, to compile mod_wsgi 3.4 for Python 2.6 on a
RedHat-like Linux::

    sudo yum install -y httpd httpd-devel
    wget https://modwsgi.googlecode.com/files/mod_wsgi-3.4.tar.gz
    tar xzf mod_wsgi-3.4.tar.gz
    cd mod_wsgi-3.4
    ./configure --with-python=/some/directory/python
    make
    make install

To ease testing of several matrix combinations, copy the resulting
``mod_wsgi.so`` to a safe place.

Start mongod
............

Start a standalone listening on port 27017, or a replica set with a member
listening on port 27017.

Configure Apache
................

Copy the appropriate version of ``mod_wsgi.so`` into Apache's modules
directory. Start Apache with the ``mod_wsgi_test.conf`` in this directory.

Run the test
------------

Run the included ``test_client.py`` script::

    python test/mod_wsgi_test/test_client.py -n 2500 -t 100 parallel \
         http://localhost/${WORKSPACE}

...where the "n" argument is the total number of requests to make to Apache,
and "t" specifies the number of threads. ``WORKSPACE`` is the location of
the PyMongo checkout.

Run this script again with different arguments to make serial requests::

    python test/mod_wsgi_test/test_client.py -n 25000 serial \
        http://localhost/${WORKSPACE}

The ``test_client.py`` script merely makes HTTP requests to Apache. Its
exit code is non-zero if any of its requests fails, for example with an
HTTP 500.

The core of the test is in the WSGI script, ``mod_wsgi_test.wsgi`.
This script inserts some documents into MongoDB at startup, then queries
documents for each HTTP request.

If PyMongo is leaking connections and "n" is much greater than the ulimit,
the test will fail when PyMongo exhausts its file descriptors.

Automation
----------

At MongoDB, Inc. we use a Jenkins job that tests each combination in the
matrix. The job copies the appropriate version of ``mod_wsgi.so`` into
place, sets up Apache, starts a single server or replica set,
and runs ``test_client.py`` with the proper arguments.
