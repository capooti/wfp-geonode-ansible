# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "centos/7"
  config.vm.network :forwarded_port, guest: 80, host: 80
  config.vm.network :forwarded_port, guest: 8000, host: 8000
  config.vm.network :forwarded_port, guest: 8080, host: 8080
  config.vm.network :forwarded_port, guest: 5432, host: 5432
  #config.vm.network :public_network, ip: "192.168.33.27"
  config.vm.network "private_network", ip: "192.168.100.10"

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--name", "wfp-2.10", "--memory", 8000, "--cpus", 2]
  end

  config.vm.synced_folder ".", "/home/vagrant/wfp"

end
