Elasticsearch
=============

Install Elasticsearch `5.x`. Heap size is dynamically calculated based on total system memory. Plugins get installed when elasticsearch is installed initially.

Requirements
------------

`es_http_listen_port` open in firewall on host

Role Variables
--------------

**Role Settings**

|   Name       | Default Value | Description                    |
|--------------|---------------|--------------------------------|
| `es_group_name`    | es    | Name of host group that contains elasticsearch hosts. Used for `es_number_of_nodes` and to calculate `discovery.zen.minimum_master_nodes`   |
| `es_version` | `7` | Version of Elasticsearch to install. Can be a major version, such as `2` or `5`, or a specific point release, such as `2.1` or `5.1.2`. |
| `es_install_state` | `latest` | Whether or not to install the latest point release of the major version. If you want the specific `es_version` to be installed, set this to `present`. In the default configuration, the latest point release of the major version will be installed. |
| `es_number_of_nodes` | `[calculated]` | Total number of nodes in the cluster. Used for calculating number of minimum master and recovery nodes. |
| `es_number_of_no_data_nodes` | `0` | Set this to the number of no data or master only nodes in your cluster. How many nodes have no data, subtracted from number of nodes that need to be up for recovery. |
| `es_plugin_bin` | `{{ es_home }}/bin/elasticsearch-plugin` | Path to the plugin binary. |
| `es_plugins` | `[]` | List of plugins to install. As of 5.4, only official plugins are supported. |
| `es_ssl_proxy` | `False` | Whether or not an SSL proxy is used in front of Elasticsearch. Requires `nginx` role. |
| `es_http_listen_port` | `{{ es_http_port }}` | ES HTTP port when using a proxy server. This will be the port in `elasticsearch.yml`. |
| `es_disable_swap` | `True` | Whether or not to disable swap on the system.   |
| `es_gpg_key_url` | `https://artifacts.elastic.co/GPG-KEY-elasticsearch` | URL for GPG key |
| `es_repo_url` | `https://artifacts.elastic.co/packages/{{ es_major_version }}.x` | Repository URL |
| `es_update_plugins` | `False` | Whether or not to update plugins |

**Elasticsearch Sysconfig Settings**

| Name              | Default Value       | Description          |
|-------------------|---------------------|----------------------|
| `es_home` | `/usr/share/elasticsearch` | Directory where Elasticsearch is installed. |
| `es_conf_dir` | `/etc/elasticsearch` | Directory containing Elasticsearch configuration. |
| `es_data_dir` | `/var/lib/elasticsearch` | Directory containing Elasticsearch data. |
| `es_log_dir` | `/var/log/elasticsearch` | Log directory. |
| `es_pid_dir` | `/var/run/elasticsearch` | PID directory. |
| `es_heap_size` | 50% of total system memory in MB | Memory heap size for ES |
| `es_java_opts` | `undefined` | Any extra Java options |
| `es_restart_on_upgrade` | `True` |  |
| `es_gc_log_file` | `/var/log/elasticsearch/gc.log` | Garbage collection log file. |
| `es_user` | `elasticsearch` | Elasticsearch user account. |
| `es_group` | `elasticsearch` | Elasticsearch group. |
| `es_startup_sleep_time` | `5` |  |
| `es_max_open_files` | `65536` |  |
| `es_max_locked_memory` | `[unlimited or total amount of memory]` | If `es_enabl_mlockall` is `True`, then this will be set to unlimited. |
| `es_max_map_count` | `262144` |  |


**Elasticsearch.yml Settings**

| Name              | Default Value       | Description          |
|-------------------|---------------------|----------------------|
| `es_cluster_name`    | `elasticsearch` | Name of Elasticsearch cluster. |
| `es_cluster_initiaml_master_nodes` | `[see defaults/main.yml]` | List of nodes that are eligible to be masters. Defaults to list of all nodes in `es_group_name`. |
| `es_node_name` | `{{ ansible_facts.hostname }}` | Node name. |
| `es_node_master` | `True` | Whether or not a node can become a master |
| `es_node_data` | `True` | Whether or not a node can hold data |
| `es_path_data` | `{{ es_data_dir }}` |  |
| `es_path_logs` | `{{ es_log_dir }}` |  |
| `es_enable_mlockall` | `True` | Whether or not to enable locking memory |
| `es_network_host` | `["localhost", "{{ ansible_facts.default_ipv4.address }}" ]` | List of interfaces Elasticsearch should listen on. |
| `es_http_port` | `9200` | Elasticsearch HTTP port. |
| `es_transport_port`    | 9300    | Node to node communication port   |
| `es_http_max_content_length` | 100 | Max HTTP put size used in Elasticsearch nginx proxy config and `elasticsearch.yml` |
| `es_minimum_master_nodes` | `{{ ((es_number_of_nodes | int) // 2) + 1 }}` | Minimum number of master nodes that must be available before the cluster is up. |
| `es_unicast_discovery_hosts`  | `[undefined]` | A comma separated list of hosts to contact for discovery. |
| `es_discovery_seed_hosts` | `[undefined]` | A list of `host:port` to use for discovery. Only applicable in Elasticsearch >= 7. |
| `es_recovery_nodes` | `{{ es_number_of_nodes | int - es_number_of_no_data_nodes }}` |  |

Example Playbook
------------
This role works best with a few pre and post tasks as well as some cluster health checking logic

```yaml
    - name: Configure Elasticsearch nodes
      hosts: es
      sudo: yes
      serial: 1

      vars:
        es_cluster_name: "mystash"
        es_number_of_no_data_nodes: 1

      tasks:
        - name: Disable shard allocation for the cluster
          uri:
            url: http://localhost:{{ es_http_listen_port }}/_cluster/settings
            method: PUT
            body: '{"transient":{"cluster.routing.allocation.enable":"none"}}'

        - name: Speed up node recovery
          uri:
            url: http://localhost:{{ es_http_listen_port }}/_cluster/settings
            method: PUT
            body: '{"transient":{"cluster.routing.allocation.cluster_concurrent_rebalance":"16","cluster.routing.allocation.node_concurrent_recoveries":"16","cluster.routing.allocation.node_initial_primaries_recoveries":"64"}}'

        - import_role:
            name: samdoran.elasticsearch

        - name: Wait for elasticsearch node to come back up
          wait_for:
            port: "{{ es_transport_port }}"
            delay: 35

        - name: Enable shard allocation for the cluster
          uri:
            url: http://localhost:{{ es_http_listen_port }}/_cluster/settings
            method: PUT
            body: '{"transient":{"cluster.routing.allocation.enable":"all"}}'

        - name: Wait for cluster health to be green
          uri:
            url: http://localhost:{{ es_http_listen_port }}/_cluster/health
            method: GET
          register: response
          until: "response.json.status == 'green'"
          always_run: True
          retries: 72
          delay: 300

        - name: Return routing allocation settings to default
          uri:
            url: http://localhost:{{ es_http_listen_port }}/_cluster/settings
            method: PUT
            body: '{"transient":{"cluster.routing.allocation.cluster_concurrent_rebalance":"2","cluster.routing.allocation.node_concurrent_recoveries":"2","cluster.routing.allocation.node_initial_primaries_recoveries":"4"}}'
          when: ansible_facts.fqdn == play_hosts[-1]
```

You can also make `es_discovery_seed_hosts` or `es_unicast_discovery_hosts` dynamic using the following:

    es_unicast_discovery_hosts: "{{ groups[es_group_name] | map('extract', hostvars, ['ansible_facts', 'fqdn']) | difference([ansible_facts.fqdn]) list | join(',') }}"

License
-------

MIT
