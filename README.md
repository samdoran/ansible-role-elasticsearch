elasticsearch
========

Install elasticsearch. Heap size is dynamically calculated based on total system memory. Plugins get installed when elasticsearch is installed initially.

Requirements
------------

java 1.7.0_55 or 1.8.0_5

Role Variables
--------------

**es_cluster_name:**    Name of the cluster (Default: elasticsearch)

**es_transport_port:**  Node to node communication port (Default: 9300)

**es_http_port:**       HTTP port (Default: 9200)

**es_java_opts_xmn:**   ES java options new heap size (Default: 128m)

**es_java_opts_xss:**   ES java stack memory (Default: 256k)

**es_update_plugins**   Whether or not to update plugins (Default: false)


Example Playbook
------------
This role works best with a few pre and post tasks as well as some cluster health checking logic

    - name: Configure Elasticsearch nodes
      hosts: es
      sudo: yes
      serial: 1

      vars:
        uribody_true:  '{"transient":{"cluster.routing.allocation.enable":"none"}}'
        uribody_false: '{"transient":{"cluster.routing.allocation.enable":"all"}}'

      pre_tasks:
        - name: Disable shard allocation for the cluster
          uri:  url=http://localhost:{{ es_http_port }}/_cluster/settings method=PUT body='{{ uribody_true }}'
          tags: elasticsearch

      roles:
        - common
        - elasticsearch

      post_tasks:
        - name:     wait for elasticsearch node to come back up
          wait_for: port={{ es_transport_port }} delay=30
          tags:     elasticsearch

        - name: Enable shard allocation for the cluster
          uri:  url=http://localhost:{{ es_http_port }}/_cluster/settings method=PUT body='{{ uribody_false }}'
          tags: elasticsearch

        - name:     Wait for cluster health to recover
          uri:      url=http://localhost:{{ es_http_port }}/_cluster/health method=GET
          register: response
          until:    "response.json.unassigned_shards < 700"
          retries:  5
          delay:    120
          tags:     elasticsearch

License
-------

MIT