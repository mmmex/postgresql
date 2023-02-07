# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :'master' => {
        :box_name => "centos/7",
        #:public => [ {adapter: 2, name: "vboxnet0", ip: '192.168.57.10', netmask: "255.255.255.0"} ],
        :private => [
          {adapter: 2, name: "vboxnet0", ip: '192.168.57.10', netmask: "255.255.255.0"}
          # {adapter: 4, ip: '192.168.58.10', netmask: "255.255.255.0", virtualbox__intnet: "clients2-net"}
        ]
  },
  :'replica1' => {
        :box_name => "centos/7",
        :private => [
          {adapter: 2, name: "vboxnet0", ip: '192.168.57.11', netmask: "255.255.255.0"}
        ]
  },
  :'backup' => {
        :box_name => "centos/7",
        :private => [
          {adapter: 2, name: "vboxnet0", ip: '192.168.57.12', netmask: "255.255.255.0"}
        ]
  }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
        config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          if boxconfig.key?(:public)
            boxconfig[:public].each do |ipconf|
              #box.vm.network "public_network", boxconfig[:public]
              box.vm.network "private_network", ipconf
            end
          end

          boxconfig[:private].each do |ipconf|
            box.vm.network "private_network", ipconf
          end

          # config.vm.synced_folder "ansible", "/ansible", :mount_options => ["dmode=700", "fmode=700"]

          case boxname.to_s
          when "master"
            box.vm.provider :virtualbox do |vb|
              vb.customize [
                'modifyvm', :id,
                '--memory', '2048',
                '--cpus', '2',
              ]
            end
          when "replica1"
            box.vm.provider :virtualbox do |vb|
              vb.customize [
                'modifyvm', :id,
                '--memory', '2048',
                '--cpus', '2',
              ]
            end
          when "backup"
            box.vm.provider :virtualbox do |vb|
              vb.customize [
                'modifyvm', :id,
                '--memory', '1024',
                '--cpus', '1',
              ]
            end
          end
          if boxname.to_s == "backup"
            box.vm.provision "ansible" do |ansible|
            # ansible.compatibility_mode = "2.0"
            ansible.become = true
            ansible.limit = "all"
            # ansible.inventory_path = "ansible/inventory/hosts"
            # ansible.config_file = "ansible/ansible.cfg"
            # ansible.verbose = "vvvv"
            ansible.playbook = "ansible/provision.yml"
          end
        end
      end
    end
end