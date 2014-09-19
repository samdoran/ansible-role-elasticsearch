elasticsearch
========

Install elasticsearch. Heap size is dynamically calculated based on total system memory. Plugins get installed when elasticsearch is installed initially.

Requirements
------------

java 1.7.0_55 or 1.8.0_5

Role Variables
--------------

**es_http_port**       HTTP port (Default: 9200)

**es_transport_port**  Node to node communication port (Default: 9300)

**es_cluster_name**    Name of the cluster (Default: elasticsearch)

**es_multicast_ip**      Multicast address used for ES node discovery (Default: 224.2.2.4)

**es_multicast_port**    Multicast port used for ES node discovery (Default: 54328)

**es_cluster_name**     Name of elasticsearch cluster (Default: elasticsearch)

**es_number_of_shards**  Number of shards (Default: 5)

**es_number_of_replicas**  Number of replicas (Default: 1)

**es_group_name**       Name of host group that contains elasticsearch hosts. Used for `es_number_of_nodes` and to calculate `discovery.zen.minimum_master_nodes` (Default: es)

**es_heap_size**        Memory heap size for ES (Default: 50% of total system memory in MB)

**es_node_master**      Whether or not a node can become a master (Default: true)

**es_node_data**      Whether or not a node can hold data (Default: true)

**es_java_opts_xmn**   ES java options new heap size (Default: 128m)

**es_java_opts_xss**   ES java stack memory (Default: 256k)

**es_update_plugins**   Whether or not to update plugins (Default: false)

**es_disable_swap**       Whether or not to disable swap on the system (Default: true)

**es_number_of_no_data_nodes**   How many nodes have no data, subtracted from number of nodes that need to be up for recovery (es_recovery_nodes).


Example Playbook
------------
This role works best with a few pre and post tasks as well as some cluster health checking logic

    - name: Configure Elasticsearch nodes
      hosts: es
      sudo: yes
      serial: 1

      vars:
        uribody_true: '{"transient":{"cluster.routing.allocation.enable":"none"}}'
        uribody_false: '{"transient":{"cluster.routing.allocation.enable":"all"}}'

      pre_tasks:
        - name: Disable shard allocation for the cluster
          uri: url=http://localhost:{{ es_http_port }}/_cluster/settings method=PUT body='{{ uribody_true }}'
          tags: elasticsearch

      roles:
        - common
        - elasticsearch

      post_tasks:
        - name: Wait for elasticsearch node to come back up
          wait_for: port={{ es_transport_port }} delay=30
          tags: [ "elasticsearch" , "esconfig" ]

        - name: Enable shard allocation for the cluster
          uri: url=http://localhost:{{ es_http_port }}/_cluster/settings method=PUT body='{{ uribody_false }}'
          tags: [ "elasticsearch" , "esconfig" ]

        - name: Wait for cluster health to recover
          uri: url=http://localhost:{{ es_http_port }}/_cluster/health method=GET
          register: response
          until:    "response.json.unassigned_shards < 700"
          retries: 5
          delay: 120
          tags:     [ "elasticsearch" , "esconfig" ]


License
-------

MIT
