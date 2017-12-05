# ansible-docker

Ansible role to install Docker stand-alone or in swam mode on a RHEL/CentOS 7
host.

Swarm support is work in progress but it can successfully build a running
cluster.

The file containing the TLS key for accessing the Docker API is currently
not encrypted.  It could be encrypted with ansible-vault because the copy
module would decrypt it on the fy.  Unfortunately it's also currently
fetched unconditionally after that.  This should be changed.

## Stand-alone example

```yaml
- name: "Docker Setup"
  hosts: legolas-docker
  roles:
  - { role: docker-new, docker_selinux_enabled: True, docker_icc: False, docker_bip: '172.17.0.1/16', docker_ipv6: True, docker_fixed_ipv6_cidr: 'fdd0:2216:12ff:f9ab::/64' }
  tags: [ 'docker' ]
```

## Swarm example

```yaml
# First node (must be a manager)
- hosts: toolchain-docker-build.0
  vars:
    ansible_user: 'ansible'
    ansible_ssh_common_args: '-i ../secrets/ssh-ansible/id_rsa'
  roles:
   - role: docker
     docker_swarm: True
     docker_swarm_manager: True
     docker_tls_generate_if_not_exists: True
  tags: ['swarm']

# More manager nodes (pass worker-token.txt to get workers)
- hosts: toolchain-docker-build:!toolchain-docker-build.0
  vars:
    ansible_user: 'ansible'
    ansible_ssh_common_args: '-i ../secrets/ssh-ansible/id_rsa'
  roles:
   - role: docker
     docker_swarm: True
     docker_swarm_manager: True
     docker_tls_generate_if_not_exists: False
     docker_swarm_manager_ipv4_address: "{{ hostvars[groups['toolchain-docker-build.0'][0]]['ansible_default_ipv4']['address'] }}"
     docker_swarm_manager_token: "{{ lookup('file', './files/vm---toolchain-docker-build01/docker-swarm/manager-token.txt') }}"
     docker_tls_key: "./files/{{ hostvars[groups['toolchain-docker-build.0'][0]]['ansible_hostname'] }}/docker-tls/key.pem"
     docker_tls_cert: "./files/{{ hostvars[groups['toolchain-docker-build.0'][0]]['ansible_hostname'] }}/docker-tls/cert.pem"
     docker_tls_ca: "./files/{{ hostvars[groups['toolchain-docker-build.0'][0]]['ansible_hostname'] }}/docker-tls/ca.pem"
  tags: ['swarm']
```
