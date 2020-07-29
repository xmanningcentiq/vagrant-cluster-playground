# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.9.7"

$rhelProvisioningScript = <<-SCRIPT
if [[ ! -f /VAGRANT_PROVISION ]] ; then
  echo "Provisioning VM, this may take a while!" | tee -a /VAGRANT_PROVISION
  echo "" | tee -a /VAGRANT_PROVISION

  echo "Installing ansible dependencies ... " | tee -a /VAGRANT_PROVISION
  sudo dnf install -y vim python3-dnf python36 libselinux python3-libselinux >> /VAGRANT_PROVISION 2>&1
  sudo pip3 install selinux >> /VAGRANT_PROVISION 2>&1 && echo "Done"
else
  echo "Already Provisioned"
fi
SCRIPT

$suseProvisioningScript = <<-SCRIPT
if [[ ! -f /VAGRANT_PROVISION ]] ; then
  echo "Provisioning VM, this may take a while!" | tee -a /VAGRANT_PROVISION
  echo "" | tee -a /VAGRANT_PROVISION

  echo "Installing extra virtualbox guest packages ... " | tee -a /VAGRANT_PROVISION
  sudo zypper install -y virtualbox-guest-tools >> /VAGRANT_PROVISION 2>&1 && echo "Done"

  echo "Installing ansible dependencies ... " | tee -a /VAGRANT_PROVISION
  sudo zypper install -y vim python-xml python-urllib3 python3-urllib3 >> /VAGRANT_PROVISION 2>&1 && echo "Done"
else
  echo "Already Provisioned"
fi
SCRIPT

$shareScript = <<-SCRIPT
IS_VAGRANT_MOUNTED="$(mount | grep vagrant || true)"
sudo test -d /vagrant || mkdir /vagrant
grep "vboxsf" /etc/fstab || echo "vagrant /vagrant vboxsf defaults 0 0" | sudo tee -a /etc/fstab
if [[ "${IS_VAGRANT_MOUNTED}" == "" ]] ;  then
  sudo mount -t vboxsf -o uid=$(id -u vagrant),gid=$(id -g vagrant) vagrant /vagrant
fi
SCRIPT

Vagrant.configure("2") do |config|
  VAGRANT_ROOT = File.join(File.dirname(File.expand_path(__FILE__)), '.vagrant/')
  PROJECT_ROOT = File.dirname(File.expand_path(__FILE__))
  config.vm.define "vgtrhqrm" do |rhq|
      rhq.vm.hostname    = "vgtrhqrm"
      rhq.vm.box         = "centos/8"
      rhq.vm.network     "private_network", ip: "192.168.61.123"

      rhq.vm.provision "shell", inline: $rhelProvisioningScript

      rhq.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "Red Hat Cluster Quorum Node"
        vb.memory = "256"
      end
  end
  config.vm.define "vgtrhdb01" do |rh1|
      rh1.vm.hostname    = "vgtrhdb01"
      rh1.vm.box         = "centos/8"
      rh1.vm.network     "private_network", ip: "192.168.61.121"

      rh1.vm.provision "shell", inline: $rhelProvisioningScript

      rh1.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "Red Hat Cluster Node 1"
        vb.memory = "512"
      end
  end
  config.vm.define "vgtrhdb02" do |rh2|
      rh2.vm.hostname    = "vgtrhdb02"
      rh2.vm.box         = "centos/8"
      rh2.vm.network     "private_network", ip: "192.168.61.122"

      rh2.vm.provision "shell", inline: $rhelProvisioningScript

      rh2.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "Red Hat Cluster Node 2"
        vb.memory = "512"
      end
  end
  config.vm.define "vgtslqrm" do |slq|
      slq.vm.hostname    = "vgtslqrm"
      slq.vm.box         = "bento/opensuse-leap-15.2"
      slq.vm.network     "private_network", ip: "192.168.61.128"
      slq.vm.synced_folder ".", "/vagrant", disabled: true

      slq.vm.provision "shell", inline: $suseProvisioningScript
      slq.vm.provision "shell", inline: $shareScript, run: "always"
      slq.vm.provision "shell", path: ".provision/scripts/fix_fstab_uuid.sh"

      slq.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "SUSE Cluster Quorum Node"
        vb.memory = "256"
        vb.customize ['sharedfolder', 'add', :id, '--name', 'vagrant', '--hostpath', PROJECT_ROOT]
      end
  end
  config.vm.define "vgtsldb01" do |sl1|
      sl1.vm.hostname    = "vgtsldb01"
      sl1.vm.box         = "bento/opensuse-leap-15.2"
      sl1.vm.network     "private_network", ip: "192.168.61.126"
      sl1.vm.synced_folder ".", "/vagrant", disabled: true

      sl1.vm.provision "shell", inline: $suseProvisioningScript
      sl1.vm.provision "shell", inline: $shareScript, run: "always"
      sl1.vm.provision "shell", path: ".provision/scripts/fix_fstab_uuid.sh"

      sl1.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "SUSE Cluster Node 1"
        vb.memory = "512"
        vb.customize ['sharedfolder', 'add', :id, '--name', 'vagrant', '--hostpath', PROJECT_ROOT]
      end
  end
  config.vm.define "vgtsldb02" do |sl2|
      sl2.vm.hostname    = "vgtsldb02"
      sl2.vm.box         = "bento/opensuse-leap-15.2"
      sl2.vm.network     "private_network", ip: "192.168.61.127"
      sl2.vm.synced_folder ".", "/vagrant", disabled: true

      sl2.vm.provider "virtualbox" do |vb|
        vb.gui    = false
        vb.name   = "SUSE Cluster Node 2"
        vb.memory = "512"
        vb.customize ['sharedfolder', 'add', :id, '--name', 'vagrant', '--hostpath', PROJECT_ROOT]
      end

      sl2.vm.provision "shell", inline: $suseProvisioningScript
      sl2.vm.provision "shell", inline: $shareScript, run: "always"
      sl2.vm.provision "shell", path: ".provision/scripts/fix_fstab_uuid.sh"

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
