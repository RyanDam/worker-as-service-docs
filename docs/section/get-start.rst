Getting Start
=============

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

.. Note:: The framework MUST be running on **Python >= 3.6**.


Server
------

Define the server
^^^^^^^^^^^^^^^^^

Follow this example to implement your own server:

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

    if __name__ == "__main__":

        args = get_args_parser().parse_args([
            '-protocol', 'obj',
            '-model_dir', '/data1/ailabserver-models/face_service_models',
            '-model_name', 'mnet_double_10062019_tf19.pb',
            '-port_in', '8996',
            '-port_out', '8998',
            '-http_port', '8900',
            '-num_worker', '2',
            '-batch_size', '1',
            '-device_map', '-1',
            '-gpu_memory_fraction', '0.25',
            '-log_dir', '/tmp/log_dir'
        ])
        server = WKRServer(args, hardprocesser=Worker)

        # start server
        server.start()

        # join server
        server.join()

``Worker`` explain: 

- The core worker of server is ``WKRHardWorker`` class which you use to make your own ``Worker`` class.

- The basic 3 function to overide.

- ``Worker::get_env``: this is where you import your own classes. For the best practice, you must import your classes here to prevent multi process/thread problem.
- ``Worker::get_model``: this is where you initialize your model, or, any model as you like.
- ``Worker::predict``: this is the main processing loop. ``input`` is a list of raw data from client. Length of input is 0..<batch_size. You need to implement your own batching process here.

.. note:: the len of result returned after processing must be matched with the input.


``main`` function explain: 

- The core server is ``WKRServer`` which is a class where you specify for your ``Worker`` to work.

- This function will load your args and start the server for you.

Server args explain:

- ``protocol``: data transfer protocol, you can choose between ``obj`` (which support transfer python object) and ``npy`` (which only support for numpy array, but higher performance).

- ``model_dir``, ``model_name``: your model paths.

- ``port_in``, ``port_out``: ports of your server to run, this server will need 2 ports.

- ``http_port``: http port (optional). If you want to support Restful APIs and Dashboard, you have to specify this.

- ``num_worker``: number of your ``Worker`` instance to be clone.

- ``batch_size``: your refer batchsize to input to your worker predict function. The framework will try to group data from client requests to match your batch size.

- ``device_map``: device map for your ``Worker``, ``-1`` for ``cpu``, ``<gpu_id>`` for gpu. You can specify multiple gpu devices. If num_worker > len(device_map), then device will be reused; if num_worker < len(device_map), then device_map[:num_worker] will be used

- ``log_dir``: your log directory. By default, framework will log your info to ``.log`` file and errors to ``.err`` file.


Start service
^^^^^^^^^^^^^

After defining your server, run this command to start:

.. code-block:: bash

    python server.py

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

