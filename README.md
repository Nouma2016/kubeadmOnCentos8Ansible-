
## Playbook info

This playbook helps you automate the standard Kubernetes bootstrapping, CNI installation and cluster setup controlplane and workers setup.

## Supported OS

- CentOS 7
- CentOS 8
- Rocky Linux 8
- AlmaLinux 8

## Required Ansible
Ansible version required `2.10+`

## Tasks in the role

This role contains tasks to:

- Install basic packages required
- Setup standard system requirements - Disable Swap, Modify sysctl, Disable SELinux
- Install and configure a container runtime of your Choice - cri-o, Docker, Containerd (Default)
- Install the Kubernetes packages - kubelet, kubeadm and kubectl
- Configure Firewalld on Kubernetes Master and Worker nodes (Only Kubernetes <1.19 version)
- Install Calico CNI
- Cluster init
- Join Workers nodes to the cluster

## How to use this role

- Clone the Project:

```
$ git clone https://github.com/Nouma2016/kubeadmOnCentos8Ansible-.git
```

- Update your inventory, for example:

```
$ vim hosts
[k8snodes]
k8smaster
k8worker01
k8sworker02
k8worker03
k8sworker04
```

- Update variables in playbook file

```
$ vim k8s-prep.yml
- name: Setup Proxy
  hosts: k8snodes
  remote_user: root
  become: yes
  become_method: sudo
  #gather_facts: no
  vars:
    k8s_version: "1.27"                                  # Kubernetes version to be installed
    selinux_state: permissive                            # SELinux state to be set on k8s nodes
    timezone: "Africa/Nairobi"                           # Timezone to set on all nodes
    k8s_cni: calico                                      # calico, flannel
    container_runtime: containerd                             # docker, cri-o, containerd
    pod_network_cidr: "10.244.0.0/16"                   # pod subnet
    configure_firewalld: false                           # true / false (keep it false, k8s>1.19 have issues with firewalld)
    # Docker proxy support
    setup_proxy: false                                   # Set to true to configure proxy
    proxy_server: "proxy.example.com:8080"               # Proxy server address and port
    docker_proxy_exclude: "localhost,127.0.0.1"          # Adresses to exclude from proxy
  roles:
    - kubernetes-bootstrap
```

If you are using non root remote user, then set username and enable sudo:

```
become: yes
become_method: sudo
```

To enable proxy, set the value of `setup_proxy` to `true` and provide proxy details.

## Running Playbook

Once all values are updated, you can then run the playbook against your nodes.

**NOTE**: Recommended to disable. if you must enable, a pattern in hostname is required for master and worker nodes:

Check file:

```
$ vim roles/kubernetes-bootstrap/tasks/configure_firewalld.yml
....
- name: Configure firewalld on master nodes
  ansible.posix.firewalld:
    port: "{{ item }}/tcp"
    permanent: yes
    state: enabled
  with_items: '{{ k8s_master_ports }}'
  when: "'master' in ansible_hostname"

- name: Configure firewalld on worker nodes
  ansible.posix.firewalld:
    port: "{{ item }}/tcp"
    permanent: yes
    state: enabled
  with_items: '{{ k8s_worker_ports }}'
  when: ("'node' in ansible_hostname" or "'worker' in ansible_hostname")

```

If your master nodes doesn't contain `master` and nodes doesn't have `node or worker` as part of its hostname, update the file to reflect your naming pattern. My nodes are named like below:

```
k8smaster
k8sworker01
k8sworker02
....
```

Check playbook syntax to ensure no errors:

```
$ ansible-playbook --syntax-check k8s-prep.yml -i hosts

playbook: k8s-prep.yml
```


```

Playbook executed as sudo user - with password:

```
$ ansible-playbook -i hosts k8s-prep.yml --ask-pass --ask-become-pass
```

```
Execution should be successful without errors:

```
TASK [kubernetes-bootstrap : Reload firewalld] *********************************************************************************************************
changed: [k8smaster]
changed: [k8sworker01]
changed: [k8sworker02]

PLAY RECAP *********************************************************************************************************************************************
master                     : ok=33   changed=26   unreachable=0    failed=0    skipped=25   rescued=0    ignored=0
k8sworker01                    : ok=27   changed=20   unreachable=0    failed=0    skipped=30   rescued=0    ignored=0
k8sworker02                    : ok=27   changed=20   unreachable=0    failed=0    skipped=30   rescued=0    ignored=0
```
