# Wazuh-Ansible

[![Slack](https://img.shields.io/badge/slack-join-blue.svg)](https://wazuh.com/community/join-us-on-slack/)
[![Email](https://img.shields.io/badge/email-join-blue.svg)](https://groups.google.com/forum/#!forum/wazuh)
[![Documentation](https://img.shields.io/badge/docs-view-green.svg)](https://documentation.wazuh.com)
[![Documentation](https://img.shields.io/badge/web-view-green.svg)](https://wazuh.com)

These playbooks install and configure Wazuh agent, manager and indexer and dashboard.

## Branches

- `main` branch contains the latest code, be aware of possible bugs on this branch.

## Compatibility Matrix

| Wazuh version | Elastic | ODFE   |
|---------------|---------|--------|
| v4.3.0+       |   N/A   |   N/A  |
| v4.2.6        | 7.10.2  | 1.13.2 |
| v4.2.5        | 7.10.2  | 1.13.2 |
| v4.2.4        | 7.10.2  | 1.13.2 |
| v4.2.3        | 7.10.2  | 1.13.2 |
| v4.2.2        | 7.10.2  | 1.13.2 |
| v4.2.1        | 7.10.2  | 1.13.2 |
| v4.2.0        | 7.10.2  | 1.13.2 |
| v4.1.5        | 7.10.2  | 1.13.2 |
| v4.1.4        | 7.10.0  | 1.12.0 |
| v4.1.3        | 7.10.0  | 1.12.0 |
| v4.1.2        | 7.10.0  | 1.12.0 |
| v4.1.1        | 7.10.0  | 1.12.0 |

## Documentation

- [Wazuh Ansible documentation](https://documentation.wazuh.com/current/deploying-with-ansible/index.html)
- [Full documentation](http://documentation.wazuh.com)

## Directory structure

    ├── wazuh-ansible
    │ ├── roles
    │ │ ├── wazuh
    │ │ │ ├── ansible-filebeat-oss
    │ │ │ ├── ansible-wazuh-manager
    │ │ │ ├── ansible-wazuh-agent
    │ │ │ ├── wazuh-dashboard
    │ │ │ ├── wazuh-indexer
    │ │
    │ │ ├── ansible-galaxy
    │ │ │ ├── meta
    │
    │ ├── playbooks
    │ │ ├── wazuh-agent.yml
    │ │ ├── wazuh-dashboard.yml
    │ │ ├── wazuh-indexer.yml
    │ │ ├── wazuh-manager-oss.yml
    | | ├── wazuh-production-ready
    │ │ ├── wazuh-single.yml
    │
    │ ├── README.md
    │ ├── VERSION.json
    │ ├── CHANGELOG.md

## Example: Wazuh all-in-one deployment on 3 servers.

### Playbook

The playbok under here will deploy wazuh all-in-one deployment on 3 defined servers. So each server will have indexer,server in a cluster.

```yaml
---
# =====================================================
# Wazuh Indexer Cluster
# =====================================================
# IMPORTANT NODE NAMES MUST CHANGE ACCORDINGLY
- hosts: wi_cluster
  strategy: free
  become: yes
  become_user: root
  roles:
    - role: ../roles/wazuh/wazuh-indexer
      indexer_network_host: "{{ private_ip }}"
  vars:
    indexer_cluster_nodes:
      - "{{ hostvars.wi1.private_ip }}"
      - "{{ hostvars.wi2.private_ip }}"
      - "{{ hostvars.wi3.private_ip }}"

    indexer_discovery_nodes:
      - "{{ hostvars.wi1.private_ip }}"
      - "{{ hostvars.wi2.private_ip }}"
      - "{{ hostvars.wi3.private_ip }}"

    indexer_node_master: true
    indexer_admin_password: changeme
    indexer_custom_user: ""
    indexer_version: "4.12.0" # Change version
    indexer_node_name: "indexer-{{ hostvars[inventory_hostname].node_id }}" # the front of the node name must be changed if you are using different name

    wazuh_certificates:
      ca: "root-ca.pem"
      cert: "indexer-{{ hostvars[inventory_hostname].node_id }}.pem" # the front of the node name must be changed if you are using different name
      key: "indexer-{{ hostvars[inventory_hostname].node_id }}-key.pem" # the front of the node name must be changed if you are using different name

    instances:
      wazuh-indexer-1:
        name: indexer-1
        ip: "{{ hostvars.wi1.private_ip }}"
        role: indexer

      wazuh-indexer-2:
        name: indexer-2
        ip: "{{ hostvars.wi2.private_ip }}"
        role: indexer

      wazuh-indexer-3:
        name: indexer-3
        ip: "{{ hostvars.wi3.private_ip }}"
        role: indexer

      wazuh-server-1:
        name: server-1
        ip: "{{ hostvars.wi1.private_ip }}"
        role: wazuh
        node_type: master

      wazuh-server-2:
        name: server-2
        ip: "{{ hostvars.wi2.private_ip }}"
        role: wazuh
        node_type: worker

      wazuh-server-3:
        name: server-3
        ip: "{{ hostvars.wi3.private_ip }}"
        role: wazuh
        node_type: worker

      wazuh-dashboard-1:
        name: dashboard-1
        ip: "{{ hostvars.wi1.private_ip }}"
        role: dashboard


# =====================================================
# Wazuh Managers (Master + Workers)
# =====================================================
- hosts: manager
  become: yes
  become_user: root
  roles:
    - role: "../roles/wazuh/ansible-wazuh-manager"
    - role: "../roles/wazuh/ansible-filebeat-oss"
  vars:
    filebeat_output_indexer_hosts:
      - "{{ hostvars.wi1.private_ip }}"
      - "{{ hostvars.wi2.private_ip }}"
      - "{{ hostvars.wi3.private_ip }}"

    filebeat_node_name: "server-{{ hostvars[inventory_hostname].node_id }}" # the front of the node name must be changed if you are using different name
    wazuh_manager_version: 4.12.0 # Set the version
    wazuh_certificates:
      ca: "root-ca.pem"
      cert: "server-{{ hostvars[inventory_hostname].node_id }}.pem" # the front of the node name must be changed if you are using different name
      key: "server-{{ hostvars[inventory_hostname].node_id }}-key.pem" # the front of the node name must be changed if you are using different name

    wazuh_manager_config:
      connection:
        - type: secure
          port: 1514
          protocol: tcp
          queue_size: 131072

      api:
        https: yes

      cluster:
        disable: "no"
        node_name: "{{ 'server-1' if inventory_hostname == 'wi1' else 'server-' + hostvars[inventory_hostname].node_id | string }}" # the front of the node name must be changed if you are using different name
        node_type: "{{ 'master' if inventory_hostname == 'wi1' else 'worker' }}"
        key: c98b62a9b6169ac5f67dae55ae4a9088
        nodes:
          - "{{ hostvars.wi1.private_ip }}"
          - "{{ hostvars.wi2.private_ip }}"
          - "{{ hostvars.wi3.private_ip }}"


# =====================================================
# Wazuh Dashboard
# =====================================================
- hosts: dashboard
  become: yes
  become_user: root
  roles:
    - role: "../roles/wazuh/wazuh-dashboard"
      tags: ['wazuh-dashboard']
  vars:
    dashboard_version: 4.12.0 # Set version
    dashboard_node_name: dashboard-1 # Change node name accordingly
    indexer_network_host: "{{ hostvars.wi1.private_ip }}"
    indexer_cluster_nodes:
      - "{{ hostvars.wi1.private_ip }}"
      - "{{ hostvars.wi2.private_ip }}"
      - "{{ hostvars.wi3.private_ip }}"
    ansible_shell_allow_world_readable_temp: true

```

### Inventory file

- The `ansible_host` variable should contain the `address/FQDN` used to gather facts and provision each node.
- The `private_ip` variable should contain the `address/FQDN` used for the internal cluster communications.
- Whether the environment is located in a local subnet, `ansible_host` and `private_ip` variables should match.
- The ssh credentials used by Ansible during the provision can be specified in this file too. Another option is including them directly on the playbook.

Create this file under `/etc/ansible/` with filename `hosts`. Ansibile will use the defined hosts in the file to push configuration.
```ini
[wi_cluster]
192.168.110.177 node_id=1 private_ip=192.168.110.177
192.168.110.178 node_id=2 private_ip=192.168.110.178
192.168.110.179 node_id=3 private_ip=192.168.110.179

[manager]
wi1 private_ip=192.168.110.177 node_id=1
wi2 private_ip=192.168.110.178 node_id=2
wi3 private_ip=192.168.110.179 node_id=3

[dashboard]
wi1 private_ip=192.168.110.177 node_id=1

[all:vars]
ansible_ssh_private_key_file=/root/.ssh/id_rsa
ansible_ssh_extra_args='-o StrictHostKeyChecking=no'
```

### Launching the playbook

```bash
sudo ansible-playbook wazuh-3-servers.yml
```

After the playbook execution, the Wazuh UI should be reachable through `https://<dashboard_host>`

## Example: single-host environment

### Playbook

The hereunder example playbook uses the `wazuh-ansible` role to provision a single-host Wazuh environment. This architecture includes all the Wazuh and Opensearch components in a single node.

```yaml
---
# Certificates generation
  - hosts: aio
    roles:
      - role: ../roles/wazuh/wazuh-indexer
        perform_installation: false
    become: no
    #become_user: root
    vars:
      indexer_node_master: true
      instances:
        node1:
          name: node-1       # Important: must be equal to indexer_node_name.
          ip: 127.0.0.1
          role: indexer
    tags:
      - generate-certs
# Single node
  - hosts: aio
    become: yes
    become_user: root
    roles:
      - role: ../roles/wazuh/wazuh-indexer
      - role: ../roles/wazuh/ansible-wazuh-manager
      - role: ../roles/wazuh/ansible-filebeat-oss
      - role: ../roles/wazuh/wazuh-dashboard
    vars:
      single_node: true
      minimum_master_nodes: 1
      indexer_node_master: true
      indexer_network_host: 127.0.0.1
      filebeat_node_name: node-1
      filebeat_output_indexer_hosts:
      - 127.0.0.1
      instances:
        node1:
          name: node-1       # Important: must be equal to indexer_node_name.
          ip: 127.0.0.1
          role: indexer
      ansible_shell_allow_world_readable_temp: true
```

### Inventory file

```ini
[aio]
<your server host>

[all:vars]
ansible_ssh_user=vagrant
ansible_ssh_private_key_file=/path/to/ssh/key.pem
ansible_ssh_extra_args='-o StrictHostKeyChecking=no'
```

### Launching the playbook

```bash
sudo ansible-playbook wazuh-single.yml -i inventory
```

After the playbook execution, the Wazuh UI should be reachable through `https://<your server host>`

## Example: Wazuh server cluster (without Filebeat)

### Playbook

The hereunder example playbook uses the `wazuh-ansible` role to provision a Wazuh server cluster without Filebeat. This architecture includes 2 Wazuh servers distributed in two different nodes.

```yaml
---
# Wazuh cluster without Filebeat
    - hosts: manager
      roles:
        - role: "../roles/wazuh/ansible-wazuh-manager"
      become: yes
      become_user: root
      vars:
        wazuh_manager_config:
          connection:
              - type: 'secure'
                port: '1514'
                protocol: 'tcp'
                queue_size: 131072
          api:
              https: 'yes'
          cluster:
              disable: 'no'
              node_name: 'master'
              node_type: 'master'
              key: 'c98b62a9b6169ac5f67dae55ae4a9088'
              nodes:
                  - "{{ hostvars.manager.private_ip }}"
              hidden: 'no'
        wazuh_api_users:
          - username: custom-user
            password: SecretPassword1!

    - hosts: worker01
      roles:
        - role: "../roles/wazuh/ansible-wazuh-manager"
      become: yes
      become_user: root
      vars:
        wazuh_manager_config:
          connection:
              - type: 'secure'
                port: '1514'
                protocol: 'tcp'
                queue_size: 131072
          api:
              https: 'yes'
          cluster:
              disable: 'no'
              node_name: 'worker_01'
              node_type: 'worker'
              key: 'c98b62a9b6169ac5f67dae55ae4a9088'
              nodes:
                  - "{{ hostvars.manager.private_ip }}"
              hidden: 'no'
```

### Inventory file

```ini
[manager]
<your manager master server host>

[worker01]
<your manager worker01 server host>

[all:vars]
ansible_ssh_user=vagrant
ansible_ssh_private_key_file=/path/to/ssh/key.pem
ansible_ssh_extra_args='-o StrictHostKeyChecking=no'
```

### Adding additional workers

Add the following block at the end of the playbook

```yaml
    - hosts: worker02
      roles:
        - role: "../roles/wazuh/ansible-wazuh-manager"
      become: yes
      become_user: root
      vars:
        wazuh_manager_config:
          connection:
              - type: 'secure'
                port: '1514'
                protocol: 'tcp'
                queue_size: 131072
          api:
              https: 'yes'
          cluster:
              disable: 'no'
              node_name: 'worker_02'
              node_type: 'worker'
              key: 'c98b62a9b6169ac5f67dae55ae4a9088'
              nodes:
                  - "{{ hostvars.manager.private_ip }}"
              hidden: 'no'
```

NOTE: `hosts` and `wazuh_manager_config.cluster_node_name` are the only parameters that differ from the `worker01` configuration.

Add the following lines to the inventory file:

```ini
[worker02]
<your manager worker02 server host>
```

### Launching the playbook

```bash
sudo ansible-playbook wazuh-manager-oss-cluster.yml -i inventory
```

## Contribute

If you want to contribute to our repository, please fork our Github repository and submit a pull request.

If you are not familiar with Github, you can also share them through [our users mailing list](https://groups.google.com/d/forum/wazuh), to which you can subscribe by sending an email to `wazuh+subscribe@googlegroups.com`.

### Modified by Wazuh

The playbooks have been modified by Wazuh, including some specific requirements, templates and configuration to improve integration with Wazuh ecosystem.

## Credits and Thank you

Based on previous work from dj-wasabi.

https://github.com/dj-wasabi/ansible-ossec-server

## License and copyright

WAZUH
Copyright (C) 2016, Wazuh Inc.  (License GPLv2)

## Web references

- [Wazuh website](http://wazuh.com)
