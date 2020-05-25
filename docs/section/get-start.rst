Getting Started
===============

.. contents:: :local:


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

.. Note:: The framework MUST be runned on **Python >= 3.6**.


Server
------

Define the server
^^^^^^^^^^^^^^^^^

Follow this example to implement your own server (ie ``server.py``):

.. code-block:: python

    import time
    from wkr_serving.server import WKRServer, WKRHardWorker
    from wkr_serving.server.helper import get_args_parser, import_tf

    class Worker(WKRHardWorker):
        
        def get_env(self, device_id, tmp_dir):
            tf = import_tf(device_id=device_id)
            import numpy as np

            logger = self.new_logger()

            return [tf, np, logger]
        
        def get_model(self, envs, model_dir, model_name, tmp_dir):
            tf, np, logger = envs

            logger.info("Begin load model")
            try:
                # load model
                pass
            except Exception as e:
                logger.error("Load model error")

            return model, logger
        
        def predict(self, model, input):
            model, logger = model
            
            logger.info("Processing input: {}".format(input))

            start_process = time.time()
            # preprocess
            done_preprocess = time.time()
            # predict
            done_predict = time.time()

            logger.info("DONE input: {}".format(input))

            # log statistic number
            self.record_statistic({
                'preprocess': (done_preprocess-start_process)*1000/len(input),
                'predict': (done_predict-done_preprocess)*1000/len(input),
                'batchsize': len(input)
            })

            return result

``Worker`` explain: 

- The core worker of server is ``WKRHardWorker`` class which you use to make your own ``Worker`` class.

The basic 3 functions to overide:

- ``Worker::get_env``: this is where you import your own classes. For the best practice, you must import your classes here to prevent multi process/thread problem.
- ``Worker::get_model``: this is where you initialize your model, or, any model as you like.
- ``Worker::predict``: this is the main processing loop. ``input`` is a list of raw data from client. Length of input is 0..<batch_size. You need to implement your own batching process here.

.. note:: the len of result returned after processing must be matched with the input.


Start service
^^^^^^^^^^^^^

After defining your server, run this command to start:

.. code-block:: bash

    wkr-serving-start server.Worker \
    -model_dir /path/to/model \
    -model_name model.hdf5 \
    -port_in 8996 \
    -port_out 8998 \
    -http_port 8900 \
    -num_worker 2 \
    -batch_size 1 \
    -device_map -1 \
    -gpu_memory_fraction 0.25 \
    -log_dir /tmp/log_dir

Script explain: 

- The core server is ``WKRServer`` which is a class where you specify for your ``Worker`` to work. 

- Assuming your ``Worker`` is defined in ``server.py``, This cli will load your ``Worker`` and start the server for you.

Server args explain:

- ``protocol``: data transfer protocol, you can choose between ``obj`` (which support transfer python object) and ``npy`` (which only support for numpy array, but higher performance).

- ``model_dir``, ``model_name``: your model paths.

- ``port_in``, ``port_out``: ports of your server to run, this server will need 2 ports.

- ``http_port``: http port (optional). If you want to support Restful APIs and Dashboard, you have to specify this.

- ``num_worker``: number of your ``Worker`` instance to be clone.

- ``batch_size``: your refer batchsize to input to your worker predict function. The framework will try to group data from client requests to match your batch size.

- ``device_map``: device map for your ``Worker``, ``-1`` for ``cpu``, ``<gpu_id>`` for gpu. You can specify multiple gpu devices. If num_worker > len(device_map), then device will be reused; if num_worker < len(device_map), then device_map[:num_worker] will be used

- ``log_dir``: your log directory. By default, framework will log your info to ``.log`` file and errors to ``.err`` file.


Generate production script
^^^^^^^^^^^^^^^^^^^^^^^^^^

For generating process managing script for production environment, run this script:

.. code-block:: bash

    wkr-serving-make server.Worker \
    -model_dir /path/to/model \
    -model_name model.hdf5 \
    -port_in 8996 \
    -port_out 8998 \
    -http_port 8900 \
    -num_worker 2 \
    -batch_size 1 \
    -device_map -1 \
    -gpu_memory_fraction 0.25 \
    -log_dir /tmp/log_dir \
    -name YOUR_PRODUCTION_SERVICE_NAME > run_script.sh

Script explain: 

- You need to run ``wkr-serving-make`` instead of ``wkr-serving-start`` to generate production script.
- After you run that command. `run_script.sh` will be generated to the current directory.

.. note:: You only need to run this script once, after that, use the generated ``run_script.sh`` to control your service.

1. To start service:

.. code-block:: bash

    sh run_script.sh start

.. note:: After you start your service, a ``YOUR_PRODUCTION_SERVICE_NAME.pid`` file will be created which contain your service process id. Also, your std printing will be output to ./logs/std.log

1. To stop service (safe way):

.. code-block:: bash

    sh run_script.sh stop

1. To stop service (forced way):

.. code-block:: bash

    sh run_script.sh stopf

Client
------

Calling service
^^^^^^^^^^^^^^^

.. code-block:: python

    import numpy as np
    from wkr_serving.client import WKRClient

    if __name__ == "__main__":
        client = WKRClient(ip='0.0.0.0', port=8996, port_out=8998, check_version=False)
        input = np.zeros((5,5))
        output = client.encode(input)

Started by creating a ``WKRClient``, you have to specify your server ``ip``, ``port``, ``port_out``. To send request to server, call ``encode`` function of your client.

.. note:: Your input must be an atom input, which means you *dont* encode a list of your input. The server will handle batching for you automatically.

Decentralize Client
-------------------

Define the server
^^^^^^^^^^^^^^^^^

Follow this example to implement your own server (ie ``decentralize.py``):

.. code-block:: python

    import os
    import sys
    import time
    import requests

    from wkr_serving.client import WKRWorker, WKRDecentralizeCentral
    from wkr_serving.client import WKRClient
    from wkr_serving.client.helper import RedisHandler

    class ProcessingModel(WKRWorker):

        # Create connection to service
        def get_model(self, ip, port, port_out):
            REDIS_IP = '10.40.34.14'
            REDIS_PORT = 12345
            REDIS_PASS = None
            QUEUE_KEY = 'EXAMPLE_QUEUE'
            redis_client = RedisHandler(QUEUE_KEY, REDIS_IP, REDIS_PORT, password=REDIS_PASS)
            return WKRClient(ip=ip, port=port, port_out=port_out, ignore_all_checks=True), redis_client

        # Do the work loop, just 1 job at a time for efficiency
        def do_work(self, model, logger):

            model, redis_client = model

            try:
                # Step 1: get data, ex: from redis
                input = redis_client.pop()
                if input is not None:
                    # Step 2: model.encode
                    result = model.encode(input)
                    # Step 3: push result
                    status = request.post(PUSH_API, json=result)

                    if status.status_code == 200:
                        logger.info('DONE job with input: {}'.format(input))
                    else:
                        raise Exception('Push result failed')
                else:
                    time.sleep(0.5) # sleep for 0.5s

            except Exception as e:
                raise Exception("{}\nTHIS IS CUSTOM EXCEPTION for input: {}".format(e, input))

        # close model connection to service
        def off_model(self, model):
            model, redis_client = model
            model.close()
            redis_client.close()

``ProcessingModel`` explain: 

- ``ProcessingModel`` is a basic block of a client (called as a processing unit). ``Decentralize`` will directly manage this.

The basic 3 functions to overide:

- ``ProcessingModel::get_model``: this is where your client is created. input of this function is the information of remote service this client will connect to, including ``ip``, ``port`` and ``port_out``. You can also create many other thing like redis connect, DB connect...

.. note:: this function will be called once when ``ProcessingModel`` is created.

- ``ProcessingModel::off_model``: all you need to do is move all the connection and deinit all the resources. This is very important as you don't want your system get unstable overtime.

.. note:: this function will be called once when ``ProcessingModel`` is destroyed.

- ``ProcessingModel::do_work``: this is where the main work happened. You need to implement your own logic here.

.. note:: this function will be put in a loop, so this function will likely to process only 1 request per loop.

Start decentralize
^^^^^^^^^^^^^^^^^^

After defining your decentralize, run this command to start:

.. code-block:: bash

    wkr-decentral-start decentralize.ProcessingModel \
    -port 21324 \
    -port_out 21326 \
    -http_port 8900 \
    -num_client 24 \
    -remote_servers [[10.40.34.16, 8068, 8069], [10.40.34.14, 8068, 8069]] \
    -log_dir /tmp/log_dir

Script explain: 

- The core server is ``WKRWorker`` which is a class where you specify for your ``ProcessingModel`` to work. 

- Assuming your ``ProcessingModel`` is defined in ``decentralize.py``, This cli will load your ``ProcessingModel`` and start the server for you.

Server args explain:

- ``port``, ``port_out``: ports of your server to run, this server will need 2 ports.

- ``remote_servers``: config remote server for each group of clients. remote server is specific by ``[<ip>, <port_in>, <port_out>]``

- ``num_client``: number of client of each remote server client group.

- ``log_dir``: your log directory. By default, framework will log your info to ``.log`` file and errors to ``.err`` file.

Generate production script
^^^^^^^^^^^^^^^^^^^^^^^^^^

For generating process managing script for production environment, run this script:

.. code-block:: bash

    wkr-decentral-make decentralize.ProcessingModel \
    -port 21324 \
    -port_out 21326 \
    -http_port 8900 \
    -num_client 24 \
    -remote_servers [[10.40.34.16, 8068, 8069], [10.40.34.14, 8068, 8069]] \
    -log_dir /tmp/log_dir
    -name YOUR_DECENTRALIZE_SERVICE_NAME > run_script.sh

Script explain: 

- You need to run ``wkr-decentral-make`` instead of ``wkr-decentral-start`` to generate production script.
- After you run that command. `run_script.sh` will be generated to the current directory.

.. note:: You only need to run this script once, after that, use the generated ``run_script.sh`` to control your service.

1. To start service:

.. code-block:: bash

    sh run_script.sh start

.. note:: After you start your service, a ``YOUR_DECENTRALIZE_SERVICE_NAME.pid`` file will be created which contain your service process id. Also, your std printing will be output to ./logs/std.log

1. To stop service (safe way):

.. code-block:: bash

    sh run_script.sh stop

1. To stop service (forced way):

.. code-block:: bash

    sh run_script.sh stopf
