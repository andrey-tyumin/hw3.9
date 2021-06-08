# -*- mode: ruby -*-
# vi: set ft=ruby :


Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/focal64"

  config.vm.provider "virtualbox" do |v|
    v.memory = 1536
    v.cpus = 1
  end

  config.vm.define "vault" do |vault|
   config.vm.network "forwarded_port", guest: 8200, host: 8200
   config.vm.provision "shell" do |s|
	s.inline = "curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -;"\
	"sudo apt-add-repository \"deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main\";"\
	"sudo apt update;"\
	"sudo apt install vault;"\
	"sudo apt -y install jq;"\
	"sudo apt -y install nginx"
   end
  end

end
