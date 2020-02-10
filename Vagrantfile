# coding: utf-8
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define 'server1' do |machine|
    machine.vm.box = "generic/centos8"
    machine.vm.hostname = 'server1'
    machine.vm.network "private_network", ip: "172.20.1.250"
    #machine.vm.network "public_network",  ip: "192.168.1.250", bridge: "en0: Ethernet"
    machine.vm.provider "virtualbox" do |vbox|
      vbox.gui = false        
      vbox.cpus = 1
      vbox.memory = 2048
    end
    machine.vm.synced_folder ".", "/vagrant", owner: "vagrant",
      group: "vagrant", mount_options: ["dmode=700", "fmode=700"]
    
    machine.vm.provision "ansible_local" do |ansible|
      ansible.playbook       = "playbook/install_bind.yml"
      ansible.version        = "latest"
      ansible.verbose        = false
      ansible.install        = true
      ansible.limit          = "server1"
      ansible.inventory_path = "playbook/bind/hosts"
    end
  end
end
