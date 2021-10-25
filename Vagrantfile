# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"
N = 2

# Install vagrant-disksize to allow resizing the vagrant box disk.
unless Vagrant.has_plugin?("vagrant-disksize")
  raise  Vagrant::Errors::VagrantError.new, "vagrant-disksize plugin is missing. Please install it using 'vagrant plugin install vagrant-disksize' and rerun 'vagrant up'"
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "geerlingguy/centos7"
  config.disksize.size = '80GB'
  config.ssh.insert_key = false
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider :virtualbox do |v|
    v.memory = 512
    v.linked_clone = true
  end
  
  (1..N).each do |machine_id|
    config.vm.define "centOS#{machine_id}" do |machine|
      machine.vm.hostname = "centOS#{machine_id}"
      machine.vm.network "private_network", ip: "192.168.60.#{3+machine_id}"

      if machine_id == N
        machine.vm.provision :ansible do |ansible|
          ansible.inventory_path = "inventory/inventory.ini"
          ansible.playbook = "configure.yml"
          ansible.limit = "all"
          ansible.become = true
        end
      end
    end
  end  
end
