Using WKRServer
================

.. contents:: :local:

Installation
------------

The best way to install the server is via pip. Note that the server can be installed separately from WKRClient or even on a different machine:

.. highlight:: bash
.. code-block:: bash

    pip install bert-serving-server

.. warning:: The server MUST be running on **Python >= 3.5** with **Tensorflow >= 1.10** (*one-point-ten*). Again, the server does not support Python 2!

Command Line Interface
----------------------

Once installed, you can use the command line interface to start a bert server:

.. highlight:: bash
.. code-block:: bash

    bert-serving-server -model_dir /uncased_bert_model -num_worker 4

Server-side API
---------------

Server-side is a CLI ``bert-serving-start``, you can get the latest
usage via:

.. code:: bash

   bert-serving-start --help
