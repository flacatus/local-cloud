# -*- mode: ruby -*-
# vi: set ft=ruby :

# Nodes:
#  controller-01    192.168.100.10
#  compute-01       192.168.100.13
#  openstack-client 192.168.100.99

# Interfaces
# eth0 - nat (used by VMware/VirtualBox)
# eth1 - br-mgmt (Container) 172.29.236.0/24
# eth2 - br-vlan (Neutron VLAN network) 0.0.0.0/0
# eth3 - host / API 192.168.100.0/24
# eth4 - br-vxlan (Neutron VXLAN Tunnel network) 172.29.240.0/24

nodes = {
    'compute'  => [1, 13],
    'controller' => [1, 10],
    'openstack-client' => [1,99]
}

Vagrant.configure("2") do |config|

  if Vagrant.has_plugin?("vagrant-hostmanager")
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    config.hostmanager.manage_guest = true
  else
    raise "[-] ERROR: Please add vagrant-hostmanager plugin:  vagrant plugin install vagrant-hostmanager"
  end

  # Defaults (VirtualBox)
  #config.vm.box = "velocity42/xenial64"
  config.vm.box = "bento/ubuntu-18.04"
  config.vm.synced_folder ".", "/vagrant"

  #Default is 2200..something, but port 2200 is used by forescout NAC agent.
  config.vm.usable_port_range = 2800..2900

  config.vm.graceful_halt_timeout = 120

  nodes.each do |prefix, (count, ip_start)|
    count.times do |i|
      if prefix == "compute" or prefix == "controller"
        hostname = "%s-%02d" % [prefix, (i+1)]
      else
        hostname = "%s" % [prefix, (i+1)]
      end

      config.ssh.insert_key = false

      config.vm.define "#{hostname}" do |box|
        box.vm.hostname = "#{hostname}.cook.book"
        box.vm.network :private_network, ip: "172.29.236.#{ip_start+i}", :netmask => "255.255.255.0"
        box.vm.network :private_network, ip: "10.10.0.#{ip_start+i}", :netmask => "255.255.255.0"
      	box.vm.network :private_network, ip: "192.168.100.#{ip_start+i}", :netmask => "255.255.255.0"
      	box.vm.network :private_network, ip: "172.29.240.#{ip_start+i}", :netmask => "255.255.255.0"

	      box.vm.provision :shell, :path => "hosts.sh"

        # Order is important - this is the last "prefix" (vm) to load up, so execute last
        if hostname == "openstack-client"

        	box.vm.provision :shell, :path => "hosts.sh"

          box.vm.provision :shell, :path => "scripts/install-ansible.sh"

          box.vm.provision :ansible_local do |ansible|
            ansible.install = false
            ansible.provisioning_path = "/vagrant"
            # Disable default limit to connect to all the machines
            ansible.limit = "all"
            ansible.playbook = "playbooks/deploy-ssh-keys.yml"
            ansible.inventory_path = "playbooks/hosts.ini"
            ansible.extra_vars = { ansible_user: "vagrant", ansible_ssh_pass: "vagrant" }
            ansible.become = false
          end

          box.vm.provision :ansible_local do |ansible|
            ansible.install = false
            # Disable default limit to connect to all the machines
            ansible.compatibility_mode = "2.0"
            ansible.limit = "all"
            ansible.playbook = "install-openstack.yml"
            ansible.extra_vars = { ansible_ssh_user: 'vagrant' }
            ansible.inventory_path = "playbooks/hosts.ini"
            ansible.become = true
          end

          box.vm.provision :shell, :path => "fetch-openrc-from-utility.sh"

          box.vm.provision :shell, :path => "openstack-client.sh"

        end

        # Otherwise using VirtualBox
        box.vm.provider :virtualbox do |vbox|
          vbox.name = "#{hostname}"
          # Defaults
          vbox.linked_clone = true if Vagrant::VERSION =~ /^1.8/
          vbox.customize ["modifyvm", :id, "--memory", 4096]
          vbox.customize ["modifyvm", :id, "--cpus", 1]
          if prefix == "controller"
            vbox.customize ["modifyvm", :id, "--memory", 8192]
            vbox.customize ["modifyvm", :id, "--cpus", 3]
          end
          if prefix == "compute"
            vbox.customize ["modifyvm", :id, "--memory", 82192]
            vbox.customize ["modifyvm", :id, "--cpus", 2]
          end
          vbox.customize ["modifyvm", :id, "--nicpromisc1", "allow-all"]
          vbox.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
          vbox.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
          vbox.customize ["modifyvm", :id, "--nicpromisc4", "allow-all"]
          vbox.customize ["modifyvm", :id, "--nicpromisc5", "allow-all"]
        end
      end
    end
  end
end
