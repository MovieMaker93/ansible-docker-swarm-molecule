![example workflow](https://github.com/MovieMaker93/ansible-docker-swarm-molecule/actions/workflows/ci.yml/badge.svg)

# ansible-docker-swarm-molecule

This project aims to spin up 2 VMs centos-7 with Vagrant and Virtualbox.   
Ansible is used to configure certificates, docker, secure docker daemon, and create a docker swarm on the virtual machines defined in the inventory.   
The docker daemon configuration guarantees at least 40GB of space, if accessible, for the docker partition.

## Prerequisites

1. Vagrant 2.2.18
2. Virtualbox 6.1
3. Ansible 2.9.6 and Ansible Core 2.11 
4. Import the collection defined in the **requirement.yml** file

For importing the collection run:  
```
ansible-galaxy install -r requirements.yml
```
  
## How to Test

The **Vagrant** file represents the configuration of the 2VMs centos7. 
In the root directory launch the command:   
```
vagrant up 
```
This command allows you to spin up the VMs on **Virtualbox** and after that will also start the ansible provisioner defined here:  
```

 machine.vm.provision :ansible do |ansible|
          ansible.inventory_path = "inventory/inventory.ini"
          ansible.playbook = "configure.yml"
          ansible.limit = "all"
          ansible.become = true
        end
```

In case you want to run only the provisioner (you have already on the VMs), just run the command:  
```
vagrant provision
```

## Ansible Configuration
### Project Structure
The core of ansible configuration is defined inside **configure.yml**:  
```yml
---
- import_playbook: playbooks/docker-certificate/main.yml
- import_playbook: playbooks/docker/main.yml
- import_playbook: playbooks/docker-certificate/manager-certificate.yml
- import_playbook: playbooks/docker-swarm/main.yml
- import_playbook: playbooks/docker-example-app-swarm/deploy_stack_playbook.yml
```
This file imports all the custom playbooks that are located in the **playbooks** folder.  
All the roles are inside the folder **roles**.  
The project uses three roles:
1. geerlingguy.docker
2. geerlingguy.pip
3. geerlingguy.firewall

The **geerlingguy.docker** has been customized to meet my requirements, so I have put the roles inside a custom directory and not install directly via Galaxy.  
Also I have defined the **roles_path** and the **inventory** directory inside the **ansible.cfg** file.  

### Docker certficate
First of all, ansible will run the docker-certificate playbook for securing docker via TLS encryption.   
There is a pre-task configuration to ensure that the OpenSSL package and all the folders required for docker have been installed.    
The playbook **docker-certificate/main.yml** generates all the server and client keys with OpenSSL for the docker daemon.    
In the end, ansible will copy the client key of your first node manager to your local machine with **docker-certificate/manager-certificate.yml** playbook.  
### Docker configuration
The second playbook will configure docker with daemon secured via **TLS**. The docker daemon configuration is specifed in the vars.yml file under the docker folder:  
```yml
docker_daemon_options:
  storage-driver: "devicemapper"
  debug: true
  storage-opts: ["dm.basesize=40G","dm.use_deferred_deletion=true","dm.use_deferred_removal=true"]
  hosts: ["fd://","{{ host_daemon_tcp }}"]
  tls: true
  tlscert: "/etc/docker/server-cert.pem"
  tlskey: "/etc/docker/server-key.pem"
  tlscacert: "/etc/docker/ca.pem"
  tlsverify: true
```
The **storage.opts** ensures a base size for docker of at least 40GB, then the **tls** configuration defines the key, and the hosts of docker daemon (you can contact the daemon on the machine itself or via HTTPS from outside passing the proper client key).  
To contact docker daemon manager from your local machine, you have to configure:  

``` EXPORT DOCKER_TLS_VERIFY = 1 ```  
``` EXPORT DOCKER_HOST = tcp://<ip_of_your_first_manager>:2376 ```  

By default, the previous playbook will copy the client.key in the **~/.docker**, so you can run all the docker commands without specifying the exact path of the client keys.  
Be sure to run ``` docker info ``` and see if you can contact the daemon without any problem.  
Also, a firewall configuration on the VMs will close all the ports not needed for our purpose.  

### Docker Swarm

This playbook configures the docker swarm.  
In the **inventory.file**:  
```
# Application servers
[docker_swarm_manager]
192.168.60.4 host_daemon_tcp="tcp://192.168.60.4:2376" dds_host=192.168.60.4
[docker_swarm_worker]
192.168.60.5 host_daemon_tcp="tcp://192.168.60.5:2376" dds_host=192.168.60.5

# Group 'multi' with all servers
[cluster:children]
docker_swarm_manager
docker_swarm_worker

# Variables that will be applied to all servers  
[cluster:vars]
ansible_user=vagrant
ansible_ssh_private_key_file=~/.vagrant.d/insecure_private_key
```
There is a naming convention used for the docker swarm configuration.  
The **docker_swarm_manager** defines all the manager nodes. Instead, the **docker_swarm_worker** defines all the worker nodes.   
By default, the first manager node will be the leader of the cluster.  
In the playbook, the creation of the docker swarm follows the official docker documentation. Be sure to install the collection mentioned in the configuration because it will install the service used in the playbook.  
The join token ensures that all the worker nodes will join the cluster securely because there is a **tls** encryption by default. You can rotate the key, but the manager node manages the CA via join token by default.  

### Docker Swarm Sample App
In the last playbook, there is a sample app for testing the docker swarm behavior.    
This app will deploy two services:  
1. **Ghost** application  
2. **Visualizer** application  
The first one is the classical ghost application for blogging.  
The second one is the Visualizer, which will show how the containers are placed among the nodes in the cluster.    
**Visualizer UI**:  
![image](/visualizer.png)  
## CI Configuration
For testing purposes, there are two configurations in the repository.   
First the **ansible-lint** configured in the **.ansible-lint** file.   
This step ensures that all the playbook files are well written (the configuration skips checking the roles directory).   
The second step is the **molecule testing** that will test via docker container the task **docker-certificate**.  

The Github action **ci.yml** configures these two jobs and ensures that they will be triggered each time new code is pushed in the repository.  
First job called **linting** will run the linting on all the playbooks inside the folder **/playbooks**.   
Second job called **molecule** will test in a docker container the creation of certificate and required packeges (docker-certificate playbook).    
Thus, every time something fails, the build will be marked as failed, and a status badge will show the status in the README file.  





