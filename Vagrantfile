# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  config.ssh.insert_key = false
  config.vm.box = "centos/7"
  config.vm.box_check_update = true

#  config.vm.provision "ansible" do |ansible|
#      ansible.playbook = "test.yml"
#      #ansible.sudo = true
#      ansible.verbose = "vvv"
#  end

  (1..3).each do |d|
    config.vm.define "vault#{d}" do |node|
      node.vm.network "private_network", ip: "192.168.33.7#{d}" # 10.7.0.21, 10.7.0.22, 10.7.0.23
      node.vm.hostname = "vault#{d}"
      config.vm.provider "virtualbox" do |v|
        v.memory = 1024
        v.cpus = 1
      end
      config.vm.provision "ansible" do |ansible|
        ansible.playbook = "test.yml"
        ansible.verbose = ""
      end
    end
  end

  config.vm.define "vault-demo" do |node|
    node.vm.hostname = "vault-demo"
    node.vm.network "private_network", ip: "192.168.33.60"
    config.vm.provider "virtualbox" do |v|
      v.memory = 1024
      v.cpus = 1
    end
  end
end