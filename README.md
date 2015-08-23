Purpose
=======

Spin up vagrant multi node environment and manage all nodes using Ansible. Upon spinning up vagrant will provision each node using Ansible to bootstrap the nodes. During the bootstrap each node will have a respective host_vars configuration file which will be updated with their eth1 address and ssh key file location. This allows you to run Ansible plays from within your HostOS or within any of your vagrant nodes.

Variable Definitions
====================
````
nodes.yml
````
Define the nodes to spin up
````
---
- name: node-1
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip: 192.168.250.101
- name: node-2
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip: 192.168.250.102
````

Provisions nodes by bootstrapping using Ansible
````
ansible/bootstrap.yml
````
Bootstrap Playbook
````
---
- hosts: all
  remote_user: vagrant
  sudo: yes
  vars:
    - update_host_vars: true
    - ssh_key_path: '../.vagrant/machines/{{ inventory_hostname }}/virtualbox/private_key'
  roles:
  tasks:
    - name: updating apt cache
      apt: update_cache=yes cache_valid_time=3600
      when: ansible_os_family == "Debian"

    - name: installing ansible pre-reqs
      apt: name={{ item }} state=present
      with_items:
        - python-pip
        - python-dev
      when: ansible_os_family == "Debian"

    - name: installing ansible
      pip: name=ansible state=present

    - name: ensuring host file exists in host_vars
      stat: path=./host_vars/{{ inventory_hostname }}
      delegate_to: localhost
      register: host_var
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: creating missing host_vars
      file: path=./host_vars/{{ inventory_hostname }} state=touch
      delegate_to: localhost
      sudo: false
      when: not host_var.stat.exists

    - name: updating ansible_ssh_port
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="^ansible_ssh_port{{ ':' }}" line="ansible_ssh_port{{ ':' }} 22"
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: updating ansible_ssh_host
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="^ansible_ssh_host{{ ':' }}" line="ansible_ssh_host{{ ':' }} {{ ansible_eth1.ipv4.address }}"
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: updating ansible_ssh_key
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="^ansible_ssh_private_key_file{{ ':' }}" line="ansible_ssh_private_key_file{{ ':' }} {{ ssh_key_path }}"
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: ensuring host_vars is yaml formatted
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="---" line="---" insertbefore=BOF
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars
````

Usage
=====
````
git clone https://github.com/mrlesmithjr/vagrant-ansible-template.git
cd vagrant-ansible-template
````
Update nodes.yml
````
vagrant up
````

License
-------

BSD

Author Information
------------------

Larry Smith Jr.
- @mrlesmithjr
- http://everythingshouldbevirtual.com
- mrlesmithjr [at] gmail.com