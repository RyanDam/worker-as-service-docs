Worker-as-service documentation
===============================

``worker-as-service`` is a distributed computing framework developed for the uses of Zalo/AILab. This framework is heavily based on ``bert-as-service``

Installation
------------

As default, this framework is installed on all the server of AILab for python environment ``dl-py3`` and ``dl-py3-stg``.

To manually install the framework:

- step 1: clone this project to local machine

- step 2: navigate to ``worker-as-service/server``

.. code-block:: bash

    pip install -e .

- step 3: navigate to ``worker-as-service/client``

.. code-block:: bash

    pip install -e .

- step 4: read example in ``worker-as-service/example`` for custom server

.. Note:: The framework MUST be running on **Python >= 3.6**.

Table of Content
----------------

.. toctree::
   :maxdepth: 2

   section/what-is-it
   section/concept
   section/get-start
   tutorial/index
   source/client
   source/server
   source/decentralize

Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

