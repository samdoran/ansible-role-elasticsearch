# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "chef/centos-6.6"

  # config.vm.provider :virtualbox do |vm|
  #   vm.name = "es"
  #   vm.memory = 2048
  #   vm.cpus = 1
  # end

  (1..3).each do |i|
    config.vm.define "estest-#{i}" do |node|
      node.vm.box = "chef/centos-6.6"
      node.vm.provider :virtualbox do |vm|
        vm.name = "estest-#{i}"
        vm.memory = 2048
        vm.cpus = 1
      end
      node.vm.hostname = "estest-#{i}"
      node.vm.network "private_network", ip: "10.77.19.1#{i}"
    end
  end

  # config.vm.hostname = "estest-1"
  # config.vm.network "private_network", ip: "10.77.19.10"

  # config.ssh.private_key_path = [ "~/.ssh/id_rsa" , "~/.vagrant.d/insecure_private_key" ]

  config.vm.provision "file", source: "~/.ssh/authorized_keys", destination: "/home/vagrant/.ssh/authorized_keys"
  config.vm.provision "shell", inline: <<-SHELL
    sudo yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm
    sudo yum -y install python-httplib2 libselinux-python
  SHELL

end
