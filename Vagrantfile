# --- About ---
# Configuration of 4 managed nodes + 1 control node

# --- Good to know ---
# Password for vagrant user is "vagrant"

# --- IPs ---
# controller 192.168.101.100
# managedX 192.168.101.(100 + X)

# --- Accessing the nodes --- 
# Each node can be accessed by its short name - controller, managed1, managed2, managed3, managed4
# Alternatively fqdn can be used, e. g. controller.example.com, managed1.example.com

# # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # #
NODES_NUMBER = ENV['NODES_NUMBER'] = '4'
ADDITIONAL_DISK_SIZE = 1024 * 5 # 5GiB
BOX = 'generic/rhel8'
# # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # # #
ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

Vagrant.configure '2' do |config|
  config.vm.synced_folder ".", "/vagrant", type: "rsync"
  config.vm.box_check_update = false
  config.vm.provision "shell", inline: <<-INPUT
    # # # # # # BEGIN: Install python interpreter mandatory to use Ansible
    sudo yum install -y python3
    # # # # # # END
  INPUT

  config.vm.provider :libvirt do |libvirt|
    libvirt.storage_pool_name = "claudio-libvirt-pool"
    libvirt.storage :file, :size => '5G', :bus => 'virtio', :type => 'raw', :discard => 'unmap', :detect_zeroes => 'on'
    libvirt.nested = true
  end # end libvirt

  config.vm.define "controller" do |controller|
    controller.vm.box = BOX
    controller.vm.hostname = "controller"
    controller.vm.network "private_network", ip: "192.168.101.100"
    controller.vm.provider :libvirt do |libvirt|
      libvirt.cpus = 4
    end
    controller.vm.provision "shell", inline: <<-INPUT
      # # # # # # BEGIN: Define fqdn and short names
      sudo echo "127.0.0.1 localhost controller controller.example.com" > /etc/hosts
      sudo echo "127.0.1.1 controller" >> /etc/hosts
      for ((i=1; i<=#{ ENV['NODES_NUMBER'] }; i++))
      do
        sudo echo "192.168.101.10$i managed$i managed$i.example.com" >> /etc/hosts
      done
      # # # # # # END
      
      # # # # # # BEGIN: Install editing tools and repo containing ansible
      sudo yum install -y epel-release epel-next-release --nogpgcheck
      # # # # # # END    
    INPUT
  
    controller.vm.provision "shell", privileged: false, inline: <<-INPUT
      # # # # # # BEGIN: Generate public and private key pairs - id_rsa, id_rsa.pub
      if [ ! -f .ssh/id_rsa ]; then ssh-keygen -N "" -f /home/vagrant/.ssh/id_rsa.pub; fi
      # cat /home/vagrant/.ssh/id_rsa.pub > /vagrant/id_rsa.pub
      # # # # # # END
      
      # # # # # # BEGIN: Add fingerprints of all managed servers
      for ((i=1; i<=#{ ENV['NODES_NUMBER'] }; i++))
      do
        for name in {managed$i.example.com,managed$i}
        do
          export FINGERPRINT=$(ssh-keyscan -t rsa $name 2> /dev/null)
          if ! grep -Fxq "$FINGERPRINT" .ssh/known_hosts 2> /dev/null
          then
            echo $FINGERPRINT >> .ssh/known_hosts
          fi
        done
      done
      # # # # # # END
    INPUT
  end # end controller

  (1..NODES_NUMBER.to_i).each do |i|
    config.vm.define "managed#{i}" do |node|
      node.vm.box = BOX
      node.vm.hostname = "managed#{i}"
      node.vm.provider :libvirt do |libvirt|
        libvirt.cpus = 4
      end
      node.vm.network "private_network", ip: "192.168.101.#{i + 100}"
      node.vm.network "private_network", ip: "192.168.102.#{i + 100}", auto_config: false
      node.vm.provision "shell", inline: <<-INPUT
        # # # # # # BEGIN: Define fqdn and short names
        sudo echo "127.0.0.1 localhost managed#{i} managed#{i}.example.com" > /etc/hosts
        sudo echo "127.0.1.1 managed#{i}" >> /etc/hosts
        for ((i=1; i<=#{ ENV['NODES_NUMBER'] }; i++))
        do
          sudo echo "192.168.101.10$i managed$i managed$i.example.com" >> /etc/hosts
        done
        sudo echo "192.168.101.100 controller controller.example.com" >> /etc/hosts
        # # # # # # END
      INPUT

    end # end managed
  end # end loop managed
end
