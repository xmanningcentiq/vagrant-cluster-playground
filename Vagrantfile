# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.9.7"

$provisioningScript = <<-SCRIPT
if [[ ! -f /VAGRANT_PROVISION ]] ; then
  sudo zypper install -y python-xml > /VAGRANT_PROVISION 2>&1
fi
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.define "vgtrhdb01" do |rh1|
      rh1.vm.hostname    = "vgtrhdb01"
      rh1.vm.box         = "centos/7"
      rh1.vm.network     "private_network", ip: "192.168.61.121"

      rh1.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "Red Hat Cluster Node 1"
        vb.memory = "512"
      end
  end
  config.vm.define "vgtrhdb02" do |rh2|
      rh2.vm.hostname    = "vgtrhdb02"
      rh2.vm.box         = "centos/7"
      rh2.vm.network     "private_network", ip: "192.168.61.122"

      rh2.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "Red Hat Cluster Node 2"
        vb.memory = "512"
      end
  end
  config.vm.define "vgtsldb01" do |sl1|
      sl1.vm.hostname    = "vgtsldb01"
      sl1.vm.box         = "bento/opensuse-leap-15.1"
      sl1.vm.network     "private_network", ip: "192.168.61.126"
      sl1.vm.synced_folder ".", "/vagrant", type: "rsync"

      sl1.vm.provision "shell", inline: $provisioningScript

      sl1.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "SUSE Cluster Node 1"
        vb.memory = "512"
      end
  end
  config.vm.define "vgtsldb02" do |sl2|
      sl2.vm.hostname    = "vgtsldb02"
      sl2.vm.box         = "bento/opensuse-leap-15.1"
      sl2.vm.network     "private_network", ip: "192.168.61.127"
      sl2.vm.synced_folder ".", "/vagrant", type: "rsync"

      sl2.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "SUSE Cluster Node 2"
        vb.memory = "512"
      end

      sl2.vm.provision "shell", inline: $provisioningScript

      sl2.vm.provision "ansible" do |ansible|
        ansible.playbook       = "provision-cluster.yml"
        ansible.inventory_path = "inventory-vagrant.yml"
        ansible.limit          = "all"
        ansible.verbose        = true
        ansible.extra_vars     = {
          run_with_vagrant: true
        }
      end

  end
end
