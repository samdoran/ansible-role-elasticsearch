elasticsearch
========

Install elasticsearch. Heap size is dynamically calculated based on total system memory. Plugins get installed when elasticsearch is installed initially.

Requirements
------------

`es_http_listen_port` open in firewall on host

Role Variables
--------------

|   Name       | Default Value | Description                    |
|--------------|---------------|--------------------------------|
| `es_ssl_proxy`    | False    | Whether to proxy elasticsearch through SSL. Requires nginx role.   |
| `es_http_port`    | 9200    | HTTP port   |
| `es_http_listen_port`    | es_http_port    | ES HTTP port when using a proxy server. This will be the port in `elasticsearch.yml`.   |
| `es_http_max_content_length` | 100 | Max HTTP put size used in Elasticsearch nginx proxy config and `elasticsearch.yml` |
| `es_transport_port`    | 9300    | Node to node communication port   |
| `es_multicast_ip`    | 224.2.2.4    | Multicast address used for ES node discovery   |
| `es_multicast_port`    | 54328    | Multicast port used for ES node discovery   |
| `es_cluster_name`    | elasticsearch    | Name of the cluster   |
| `es_unicast_discovery_hosts`    | undefined    | A comma separated list of hosts to contact for discovery -- when defined disables unicast.   |
| `es_cluster_name`    | elasticsearch    | Name of elasticsearch cluster   |
| `es_number_of_shards`    | 5    | Number of shards   |
| `es_number_of_replicas`    | 1    | Number of replicas   |
| `es_group_name`    | es    | Name of host group that contains elasticsearch hosts. Used for `es_number_of_nodes` and to calculate `discovery.zen.minimum_master_nodes`   |
| `es_heap_size`    | 50% of total system memory in MB    | Memory heap size for ES   |
| `es_node_master`    | true    | Whether or not a node can become a master   |
| `es_node_data`    | true    | Whether or not a node can hold data   |
| `es_java_opts_xmn`    | 128m    | ES java options new heap size   |
| `es_java_opts_xss`    | 256k    | ES java stack memory   |
| `es_update_plugins`    | false    | Whether or not to update plugins   |
| `es_disable_swap`    | true    | Whether or not to disable swap on the system   |
| `es_enable_mlockall`    | 'true'    | Whether or not to enable locking memory   |
| `es_number_of_no_data_nodes`    | es_recovery_nodes    | How many nodes have no data, subtracted from number of nodes that need to be up for recovery   |
| `es_restart_on_config_change`    | True    | Only fire the `restart elasticsearch` handler if this is True. Allows updating the config without restarting the node for changing values like `minimum_master_nodes`.   |

Example Playbook
------------
This role works best with a few pre and post tasks as well as some cluster health checking logic
```yaml
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
          delay: 3
          tags: [ "elasticsearch" , "esconfig" ]

        - name: Wait for cluster health to recover
          uri: url=http://localhost:{{ es_http_port }}/_cluster/health method=GET
          register: response
          until:    "response.json.unassigned_shards < 700"
          retries: 5
          delay: 120
          tags:     [ "elasticsearch" , "esconfig" ]
```

You can also make `es_unicast_discovery_hosts` dynamic using the following:

    es_unicast_discovery_hosts:
    "{% for host in groups[es_group_name] %}
      {%- if host != ansible_fqdn %}{{ host }},{% endif %}
    {%- endfor %}"

License
-------

MIT
