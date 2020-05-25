What is it
==========

``Worker-as-service`` is a general computing framework with distributed work on multiple processes, across multiple physical machines.

.. contents:: :local:

Highlights
----------

- Low latency: Build on 0MQ framework, which provides high performance for data transmission. Raw inplement of this service archive 0.5ms latency for 35KB package.

- Horizontal scalable and distributable on different servers.

- Easy to use: can tranfer python object (dictionary, numpy array, Pillow image...), out-of-the-box support for REST APIs (used by AILab platform API).

- Many logger support: File logger, HTTP logger.

- Statistic insight with built-in dashboard.

More features: asynchronous encoding, multicasting, mix GPU & CPU workloads...

Currently usage
---------------

- TTS Engine V2.

- 9Dash platform API.

- NSFW (platform API/core workflow).

- Many others.
