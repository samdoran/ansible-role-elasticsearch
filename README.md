elasticsearch
========

Install elasticsearch. Heap size is dynamically calculated based on total system memory.

Requirements
------------

java 1.7.0_55 or 1.8.0_5

Role Variables
--------------

**es_cluster_name:**    Name of the cluster (Default: elasticsearch)

**es_transport_port:**  Node to node communication port (Default: 9300)

**es_http_port:**       HTTP port (Default: 9200)

**es_java_opts_xmn:**   ES java options new heap size (Default 128m)

**es_java_opts_xss:**   ES java stack memory (Default 256k)


Dependencies
------------

None

License
-------

MIT