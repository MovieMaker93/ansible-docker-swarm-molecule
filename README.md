# ansible-docker-swarm-molecule

This project aims to spin up 2 VMs centos-7 with Vagrant and Virtualbox.
Ansible is used to configure certficate, docker, secure docker daemon and create a docker swarm on the virtual machines defined in the inventory.
The docker daemon is configured in order to garuantee at least 40gb of space, if free, for the docker partition.

## Prerequisites

1. Vagrant
2. Virtualbox
3. Ansible 2.11 core version
4. Import the collection defined in the requirement.txt file

The **Vagrant** file is used to configure the 2VMs centos7.
'''
vagrant up 
'''
This command allow you to spin up the machine on **Virtualbox** and after that will start also the ansible provisioner defined here:
'''

 machine.vm.provision :ansible do |ansible|
          ansible.inventory_path = "inventory/inventory.ini"
          ansible.playbook = "configure.yml"
          ansible.limit = "all"
          ansible.become = true
        end
'''

In case you want to run only the provisioner just run the command 
'''
vagrant provision
'''

For importing the collection run:
'''
ansible-galaxy install -r requirements.txt
'''
## Ansible Configuration

The core of ansible configuration is defined inside the configure.yml.
Here are defined all the playbooks for configuring docker and docker swarm securely.
### Docker certficate
First of all ansible will run the docker-certficiate, because for securing docker via TLS encryption we need atleast to generate the key via a certificate authority. The role used is customized with my needs, so via opnessl it will generate the server.key the ca.pem via a trusted certificate authority generated by openssl.
After there is a configuration for storing on your local machine the client.key of the first manager configured in the inventory file.
### Docker configuration
The second playbook will configure docker with daemon secured via TLS. The docker daemon configuration is specifed in the vars.yml file under the docker folder:
'''yml
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
'''
Basically the **storage.opts** ensures a basesize for docker of atleast 40gb, then the **tls** configuration defines the key, and the hosts of docker daemon (you can contact the daemon on the machine itself or via https from outside passing the proper client key).
In order to contact docker from your local machine you have to configure:

''' EXPORT DOCKER_TLS_VERIFY = true '''
''' EXPORT DOCKER_HOST = tcp://<ip_of_your_first_nabager>:2376 '''

By default the previous playbook will copy the client.key in the **~/.docker**, so you can run all the docker commands without specifing the exact path of the client keys.
Be sure to run ''' docker info ''' and see if you can contact the daemon without any problem.
Also there is a firewall configuration on the VMs that will close all the port not needed for our purpose.

### Docker Swarm

In the last playbook there is all the docker swarm configuration.
In the inventory.file:
'''
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
'''
There is a naming convention used for the docker swarm configuration.
All the manager nodes are defined under **docker_swarm_manager** name, instead all the worker nodes are defined under **docker_swarm_worker**. By default the first manager node will be the leader of the cluster.
In the playbook there are all the steps suggested by the official docker documentation.
The join token ensures that all the worker nodes will join the cluster securely because there is a tls encryption by default. You can rotate the key, but by default the CA is managed by the manager node via join token.

## CI
For testing purpose there are two configuration in the repository.
First the **ansible-lint** configured in the .ansible-lint file. This step ensures that all the playbook files are weel written.
The second step is the molecule testing that will test via docker container the tasks defined above.
The github action configures these two steps and ensures that they will be triggered each time new code will be pushed in the repository. Every time something would fail, the build will be marked as failed.





