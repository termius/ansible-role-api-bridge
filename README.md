# Termius API Bridge Ansible Role

Termius Ansible role allows you to automate the import of hosts and groups from your Ansible Inventory.

## Setup

Termius Ansible role requires the API Bridge to import data into your vault. [Set up Bridge API](https://account.termius.com/bridges) first.

### Installation

Install the Termius role from Ansible Galaxy on your Ansible server:

```
ansible-galaxy install termius.termius
```

Create  a new playbook `termius_import.yml` to import, update, and delete your hosts from inventory:

```
- name: Import Inventory To Termius
  hosts: all
  gather_facts: false
  roles: 
    - role: termius
      vars:
        api_url: <"API_BRIDGE_URL">
```

Provide an `api_url`  for a machine where you deployed Termius API Bridge. By default `api_url` is http://localhost:8080.

## Role variables

These variables provide additional configuration for a Termius role. They should be specified in the `vars` section of your playbook.

| Variable | Default | Comment |
| --- | --- | --- |
| api_url<br>_(string)_ | "http://localhost:8080" | The address of a host where where API Bridge Docker container is running. |
| vault<br>_(string)_ | Team | The vault where you would like to add hosts and groups. |
| host_password<br>(string) |  | A password in plain text that will be attached to a host in Termius. |
| use_groups_instead_of_tags<br>_(bool)_ | true | Ansible Inventory allows specifying several groups for one host, creating a one-to-many connection between hosts and groups. Termius supports only one-to-one connection between hosts and groups. By to _false_ your hosts will be organized with tags instead of groups in Termius. |
| host_id_key<br>_(string)_ |  | `termius` role uses API Bridge, which requires every entity to have a external ID. By default, `ansible_host` is used.<br>Specify a key which will be used for fetching a custom external id for a host from inventory instead of ansible_host. |
| group_id_key<br>_(string)_ |  | `termius` role uses API Bridge, which requires every entity to have a external ID. By default, `group_name` is used.<br>Specify a key which will be used for fetching a external id for a group from inventory instead of `group_name` |

## How does this role work?

The role gathers all the necessary details from your Ansible inventory to create hosts and groups in Termius. If an Ansible host belongs to a group, this role uses the first item from the `group_names` variable to create a group in Termius. If your Ansible hosts are in multiple groups, set `use_groups_instead_of_tags` to `false` in the Termius role; this forces the role to create tags for hosts in Termius instead of groups.

For each host, the role collects `ansible_host`, `inventory_hostname`, `ansible_port`, and `ansible_user`. It assigns a host to a group if that group was created in the previous step.

Termius role uses API Bridge to push data into your Termius vault.

## Import Hosts 

Run the playbook to import all your hosts from inventory:

```
ansible-paybook termius_import.yml
```

If you want to import only a specific group or host, use the `-l` argument and specify a group.

```
ansible-paybook termius_import.yml -l webservers
```

## Update Hosts 

Specify `update` tag when you run `termius_import.yml` playbook. Please note that the update operation overrides all changes you've made to the host in Termius.

```
ansible-playbook termius_import.yml --tags update
```

## Delete Hosts 

There are two ways to delete hosts from your Termius vault you can run  `termius_import.yml` with `delete` tag using the following command:

```
ansible-playbook termius_import.yml --tags delete
```

Alternatively, you can create a separate playbook that will delete a task; it could be useful to avoid running the whole role for the delete operation.

```
- name: Delete host from Termius
  hosts: "{{ target_hosts }}"
  gather_facts: false
  tasks:
    - include_role:
        name: termius
        tasks_from: delete_termius_host.ansible.yml
```

## Example Playbooks

Below are some sample playbooks to assist you with using the Termius Ansible role.

### Add hosts automatically

The following example updates a password on a target host and then imports this host with a new password into the DevOps vault in Termius.

```
- name: Provision new machine
  hosts: all
  become: yes
  tasks:
    - name: Update password for root
      user:
        name: root
        password: "{{ 'MySecurePlaintextPassword' | password_hash('sha512') }}"
        update_password: always

- name: Import hosts to Termius
  hosts: all
  gather_facts: false
  roles: 
    - role: termius
      vars:
        vault: DevOps 
        host_password: "MySecurePlaintextPassword"
```

### Delete hosts automatically 

The following example deletes a docker container from a host and then deletes a host from Termius.

```
- name: Remove Docker Container
  hosts: all
  gather_facts: no
  become: yes
  tasks:
    - name: Stop and remove Docker container
      docker_container:
        name: acme_app
        state: absent

- name: Delete host from Termius
  hosts: all
  gather_facts: false
  tasks:
    - include_role:
        name: termius
        tasks_from: delete_termius_host.ansible.yml
```
