Using ``pool_worker`` to predict a very long list of tasks
==========================================================

.. contents:: :local:

Offline predict a long list of task (classify gender for face images, ) is pretty common. This is how you can implement such pipeline:

``server.py``
-------------

Define the server
^^^^^^^^^^^^^^^^^

.. note:: Please check **Getting started** section for the details.

.. code:: python

    import time
    from wkr_serving.server import WKRServer, WKRHardWorker
    from wkr_serving.server.helper import get_args_parser, import_tf

    class Worker(WKRHardWorker):
        
        def get_env(self, device_id, tmp_dir):
            tf = import_tf(device_id=device_id)
            return tf
        
        def get_model(self, envs, model_dir, model_name, tmp_dir):
            tf = envs
            try:
                model = get_model(tf, model_dir, model_name)
            except Exception as e:
                raise Exception("Custom exception\n{}".format(e)
            else:
                return model

        def predict(self, model, input):
            model = model

            results = []
            for image in input:
                score = mode.predict(image)

            return results

Start server
^^^^^^^^^^^^

We will start a server with 2 worker running on CPU (device -1)

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
    -log_dir /tmp/log_dir

.. note:: ``-http_port`` is a need as we will use it for the later step

Predictor
---------

.. code:: python

    import requests
    from io import BytesIO
    
    from ailabtools.utils.common_import import *
    from ailabtools.utils.network import download_img_file
    from ailabtools.ailab_multiprocessing import pool_worker 

    def load_list_data():
        # implement your own logic here
        return []

    def predict(img_url):
        img_pil = download_img_file(img_url)
        img_byte = BytesIO()
        img_pil.save(img_byte, 'jpeg')

        server_http_api = 'http://0.0.0.0:8900/encode_img_bytes'
        r = requests.post(server_http_api, files={'img_bytes': img_byte.getvalue()})
        data_json = r.json()
        return data_json["data"]["score"]

    list_image_urls = load_list_data()

    list_results = pool_worker(predict, list_image_urls)
    
