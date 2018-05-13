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
      node.vm.provider "virtualbox" do |v|
        v.memory = 512
        v.cpus = 1
      end
      node.vm.provision "ansible" do |ansible|
        ansible.playbook = "test.yml"
        ansible.verbose = ""
      end
    end
  end

  config.vm.define "vault-demo" do |extra|
    extra.vm.hostname = "vault-demo"
    extra.vm.network "private_network", ip: "192.168.33.60"
    extra.vm.provider "virtualbox" do |v|
      v.memory = 512
      v.cpus = 1
    end
  end
  config.vm.define "postgres" do |db|
    db.vm.hostname = "postgres"
    db.vm.network "private_network", ip: "192.168.33.62"
    db.vm.provider "virtualbox" do |vp|
      vp.memory = 1024
      vp.cpus = 1
    end
    db.vm.provision "ansible" do |dbpg|
      dbpg.playbook = "pgsql.yml"
      dbpg.verbose = ""
    end
  end
end