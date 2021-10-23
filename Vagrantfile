# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

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
  
  config.vm.define "master" do |app|
    app.vm.hostname = "master"
    app.vm.network :private_network, ip: "192.168.60.4"
  end

  #config.vm.define "slave" do |app|
    # app.vm.hostname = "slave"
   #  app.vm.network :private_network, ip: "192.168.60.5"
  #end



  # Ansible provisioner.
  config.vm.provision :ansible do |ansible|
    ansible.inventory_path = "inventory/inventory"
    ansible.playbook = "configure.yml"
    ansible.limit = "all"
    ansible.become = true
  end
end
