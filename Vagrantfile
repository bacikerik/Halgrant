# -*- mode: ruby -*-
# vi: set ft=ruby :
# ---
# Define the number of worker nodes
WORKER_COUNT = 8

# Define the network settings
NETWORK_PREFIX = "172.21"
NETWORK_MASK = "255.255.240.0"
NETWORK_GATEWAY = "172.21.112.1"
NETWORK_DNS = ["8.8.8.8", "8.8.4.4"]
# Define the box name and version
BOX_NAME = "almalinux/8"
# Available versions (03/Dec/2023): 8.3.20210203, 8.3.20210222, 8.3.20210330, 8.3.20210427, 8.4.20210527, 8.4.20210724, 8.4.20211014, 8.5.20211111, 8.5.20211118, 8.5.20211208, 8.5.20220316, 8.6.20220513, 8.6.20220715, 8.6.20220802, 8.6.20220819, 8.6.20220830, 8.6.20221001, 8.7.20221112, 8.7.20230228, 8.8.20230524, 8.8.20230606, 8.9.20231125
# Check the compatibility of 'Available versions' with hyperv provider
# https://app.vagrantup.com/almalinux/boxes/8
BOX_VERSION = "8.8.20230524"
# Define the memory settings
INFRA_MEMORY = 6144
WORKER_MEMORY = 3072
# Define the number of CPUs
INFRA_CPU = 8
WORKER_CPU = 4
# Define custom settings
CUSTOM_USER = "<ChangeMe!>"
CUSTOM_PASS = "<ChangeMe!>"
# Define custom private and public keys
KEY_PRIV = "<ChangeMe!>"
KEY_PUB = "<ChangeMe!>"

Vagrant.configure("2") do |config|
  # Use hyperv provider
  config.vm.provider :hyperv
  # Use root user and password for SSH access
  config.ssh.username = "root"
  config.ssh.password = "vagrant"
  config.vm.synced_folder '.', '/vagrant', disabled: true
  # Install update hosts file
  config.vm.provision "shell", inline: <<-SHELL
    for i in $(seq 1 "#{WORKER_COUNT}"); do
      echo "172.21.125.$i node$i" | sudo tee -a /etc/hosts > /dev/null;
    done
    echo "172.21.125.47 infra" | sudo tee -a /etc/hosts > /dev/null;
  SHELL
  # set a new password to user root
  config.vm.provision "shell", inline: "echo -e 'Start123456!\\nStart123456!' | passwd root"
  # Create user ebacik
  config.vm.provision "shell", inline: <<-SHELL
    useradd -p "#{CUSTOM_PASS}" -m "#{CUSTOM_USER}"
    echo ""#{CUSTOM_USER}":"#{CUSTOM_PASS}"" | sudo chpasswd
    echo ""#{CUSTOM_USER}" ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers
    echo ""#{CUSTOM_USER}" ALL=(ALL) NOPASSWD: /bin/su -" | sudo tee -a /etc/sudoers
    echo 'alias s=\"sudo su -\"' >> /home/#{CUSTOM_USER}/.bashrc
    sudo usermod -aG root "#{CUSTOM_USER}"
  SHELL
  config.vm.provision "file", source: "#{KEY_PUB}", destination: "/tmp/id_rsa.pub"
  config.vm.provision "shell", inline: <<-SHELL 
    mkdir -p /etc/ssh/authorized-keys/
    cat /tmp/id_rsa.pub >> /etc/ssh/authorized-keys/"#{CUSTOM_USER}"
    cat /tmp/id_rsa.pub >> /root/.ssh/authorized_keys
    chown "#{CUSTOM_USER}":"#{CUSTOM_USER}" /etc/ssh/authorized-keys/"#{CUSTOM_USER}"
    chmod 600 /etc/ssh/authorized-keys/"#{CUSTOM_USER}"
    chmod 600 /root/.ssh/authorized_keys
    rm /tmp/id_rsa.pub 
  SHELL
  # Modify sshd_config
  config.vm.provision "shell", inline: <<-SHELL
    sudo sed -i 's/#AuthorizedKeysFile/AuthorizedKeysFile/g' /etc/ssh/sshd_config
    sudo sed -i 's|AuthorizedKeysFile\s*.*|AuthorizedKeysFile /etc/ssh/authorized-keys/%u .ssh/authorized_keys|' /etc/ssh/sshd_config
  SHELL
  # Restart sshd.service
  config.vm.provision "shell", inline: <<-SHELL
    sudo systemctl restart sshd.service
  SHELL
  # Use extra args for SSH
  # config.ssh.extra_args = ["-o", "StrictHostKeyChecking=no", "-t", "echo 'ebacik ALL=(ALL) NOPASSWD:ALL' | sudo tee -a /etc/sudoers"]
  # Define the infra VM
  config.vm.define "infra" do |infra|
    # Use almalinux box with version 8.6
    infra.vm.box = BOX_NAME
    infra.vm.box_version = BOX_VERSION
    # Assign memory for infra VM
    infra.vm.provider :hyperv do |hv|
      hv.memory = INFRA_MEMORY
      hv.cpus = INFRA_CPU
    end
    # Use public network with default switch for internet access on infra VM
    infra.vm.network :public_network, bridge: "Default Switch"
    # Use private network with vInternal switch for internal communication on all VMs
    infra.vm.network :private_network, bridge: "vInternal", ip: "#{NETWORK_PREFIX}.125.47"
    # Set the hostname for infra VM
    infra.vm.hostname = "infra"
    # Create /etc/sysconfig/network-scripts/ifcfg-eth1 file on infra VM
    infra.vm.provision "shell", inline: <<-SHELL
      cat > /etc/sysconfig/network-scripts/ifcfg-eth1 <<EOF
      TYPE="Ethernet"
      NAME="eth1"
      DEVICE="eth1"
      BOOTPROTO="static"
      ONBOOT="yes"
      IPADDR="#{NETWORK_PREFIX}.125.47"
      GATEWAY="#{NETWORK_GATEWAY}"
      NETMASK="#{NETWORK_MASK}"
      DNS1="#{NETWORK_DNS[0]}"
      DNS2="#{NETWORK_DNS[1]}" <<EOF
      SHELL
    # Restart network service to apply changes
    infra.vm.provision "shell", inline: <<-SHELL
      sudo systemctl restart NetworkManager
      sudo systemctl enable NetworkManager
    SHELL
    # Configure infra VM as a proxy server (optional)
    infra.vm.provision "shell", inline: <<-SHELL
      # Install squid proxy server
      sudo dnf install -y squid
      # sudo systemctl enable --now squid
      # sudo firewall-cmd --permanent --add-service=squid
      # sudo firewall-cmd --reload
      # Allow access from private network
      # sed -i "/^http_access deny all/i http_access allow #{NETWORK_PREFIX}.0.0/#{NETWORK_MASK}" /etc/squid/squid.conf
      sed -i "/^http_access deny all/i http_access allow all" /etc/squid/squid.conf
      # Restart squid service to apply changes
      systemctl enable --now squid
    SHELL
    infra.vm.provision "file", source: "#{KEY_PRIV}", destination: "/tmp/id_rsa"
    infra.vm.provision "shell", inline: <<-SHELL 
      mkdir -p /home/"#{CUSTOM_USER}"/.ssh  
      mkdir -p /root/.ssh  
      chown "#{CUSTOM_USER}":"#{CUSTOM_USER}" /home/"#{CUSTOM_USER}"/.ssh 
      chown root:root /root/.ssh 
      chmod 700 /home/"#{CUSTOM_USER}"/.ssh   
      chmod 700 /root/.ssh   
      cat /tmp/id_rsa >> /home/"#{CUSTOM_USER}"/.ssh/id_rsa
      cat /tmp/id_rsa >> /root/.ssh/id_rsa
      chmod 600 /home/"#{CUSTOM_USER}"/.ssh/id_rsa
      chmod 600 /root/.ssh/id_rsa
      chown "#{CUSTOM_USER}":"#{CUSTOM_USER}" /home/"#{CUSTOM_USER}"/.ssh/id_rsa
      chown root:root /root/.ssh/id_rsa
      rm /tmp/id_rsa
    SHELL
    infra.vm.provision "file", source: "#{KEY_PUB}", destination: "/tmp/id_rsa.pub"
    infra.vm.provision "shell", inline: <<-SHELL 
      cat /tmp/id_rsa.pub >> /home/"#{CUSTOM_USER}"/.ssh/id_rsa.pub
      cat /tmp/id_rsa.pub >> /root/.ssh/id_rsa.pub
      chmod 644 /home/"#{CUSTOM_USER}"/.ssh/id_rsa.pub
      chmod 644 /root/.ssh/id_rsa.pub
      chown "#{CUSTOM_USER}":"#{CUSTOM_USER}" /home/"#{CUSTOM_USER}"/.ssh/id_rsa.pub
      chown root:root /root/.ssh/id_rsa.pub
      rm /tmp/id_rsa.pub
    SHELL
    # install vim & tree
    infra.vm.provision "shell", inline: "sudo dnf install -y vim tree"
    # SSH client configuration to disable StrictHostKeyChecking
    # infra.vm.provision "shell", inline: <<-SHELL
    #   echo 'StrictHostKeyChecking no' | sudo tee -a /etc/ssh/ssh_config
    # SHELL
    # infra.vm.provision "file", source: "#{KEY_PRIV_PEM}", destination: "/tmp/key.pem"
    # infra.vm.provision "shell", inline: <<-SHELL 
    #   mkdir -p /home/"#{CUSTOM_USER}"/pem_key  
    #   chown "#{CUSTOM_USER}":"#{CUSTOM_USER}" /home/"#{CUSTOM_USER}"/pem_key 
    #   chmod 700 /home/"#{CUSTOM_USER}"/pem_key   
    #   cat /tmp/key.pem >> /home/"#{CUSTOM_USER}"/pem_key/key.pem
    #   chmod 600 /home/"#{CUSTOM_USER}"/pem_key/key.pem
    #   chown "#{CUSTOM_USER}":"#{CUSTOM_USER}" /home/"#{CUSTOM_USER}"/pem_key/key.pem
    #   rm /tmp/key.pem
    # SHELL
    # infra.vm.provision "file", source: "#{KEY_PUB_PEM}", destination: "/tmp/key.pem.pub"
    # infra.vm.provision "shell", inline: <<-SHELL 
    #   mkdir -p /home/"#{CUSTOM_USER}"/pem_key  
    #   chown "#{CUSTOM_USER}":"#{CUSTOM_USER}" /home/"#{CUSTOM_USER}"/pem_key 
    #   chmod 700 /home/"#{CUSTOM_USER}"/pem_key   
    #   cat /tmp/key.pem.pub >> /home/"#{CUSTOM_USER}"/pem_key/key.pem.pub
    #   chmod 644 /home/"#{CUSTOM_USER}"/pem_key/key.pem.pub
    #   chown "#{CUSTOM_USER}":"#{CUSTOM_USER}" /home/"#{CUSTOM_USER}"/pem_key/key.pem.pub
    #   # rm /tmp/key.pem.pub
    # SHELL
  end
  # Define the worker VMs using a loop
  (1..WORKER_COUNT).each do |i|
    config.vm.define "node#{i}" do |node|
      # Use almalinux box with version 8.x
      node.vm.box = BOX_NAME
      node.vm.box_version = BOX_VERSION
      # Assign memory for worker VMs
      node.vm.provider :hyperv do |hv|
        hv.memory = WORKER_MEMORY
        hv.cpus = WORKER_CPU
      end
      # Use private network with vInternal switch for internal communication on all VMs
      node.vm.network :private_network, bridge: "vInternal", ip: "#{NETWORK_PREFIX}.125.#{0 + i}"
      # Set the hostname for worker VMs
      node.vm.hostname = "node#{i}"
      # Create /etc/sysconfig/network-scripts/ifcfg-eth0 file on worker VMs
      node.vm.provision "shell", inline: <<-SHELL
        cat > /etc/sysconfig/network-scripts/ifcfg-eth0 <<EOF
        TYPE="Ethernet"
        NAME="eth0"
        DEVICE="eth0"
        BOOTPROTO="static"
        ONBOOT="yes"
        IPADDR="#{NETWORK_PREFIX}.125.#{0 + i}"
        GATEWAY="#{NETWORK_GATEWAY}"
        NETMASK="#{NETWORK_MASK}"
        DNS1="#{NETWORK_DNS[0]}"
        DNS2="#{NETWORK_DNS[1]}" <<EOF
      SHELL
      node.vm.provision "shell", inline: <<-SHELL
        cat > /etc/sysconfig/network-scripts/route-eth0 <<EOF
        default via 172.21.112.1 dev eth0 proto static metric 100
      SHELL
      # Restart network service to apply changes
      node.vm.provision "shell", inline: <<-SHELL
        sudo systemctl restart NetworkManager
        sudo systemctl enable NetworkManager
      SHELL
      # Configure worker VMs to use infra VM as a proxy server (optional)
      node.vm.provision "shell", inline: <<-SHELL
        # Set http_proxy and https_proxy environment variables
        echo "export http_proxy=http://#{NETWORK_PREFIX}.125.47:3128" >> /etc/profile.d/proxy.sh
        echo "export https_proxy=http://#{NETWORK_PREFIX}.125.47:3128" >> /etc/profile.d/proxy.sh
        # Set proxy for dnf
        echo "proxy=http://#{NETWORK_PREFIX}.125.47:3128" >> /etc/dnf/dnf.conf
      SHELL
      # Set up proxy for almalinux repo on node VMs
      node.vm.provision "shell", inline: <<-SHELL
        sed -i '/gpgkey/a proxy=http://172.21.125.47:3128' /etc/yum.repos.d/almalinux.repo
        sed -i '/gpgkey/a proxy=http://172.21.125.47:3128' /etc/yum.repos.d/almalinux-ha.repo
        sed -i '/gpgkey/a proxy=http://172.21.125.47:3128' /etc/yum.repos.d/almalinux-nfv.repo
        sed -i '/gpgkey/a proxy=http://172.21.125.47:3128' /etc/yum.repos.d/almalinux-plus.repo
        sed -i '/gpgkey/a proxy=http://172.21.125.47:3128' /etc/yum.repos.d/almalinux-powertools.repo
        sed -i '/gpgkey/a proxy=http://172.21.125.47:3128' /etc/yum.repos.d/almalinux-resilientstorage.repo
        sed -i '/gpgkey/a proxy=http://172.21.125.47:3128' /etc/yum.repos.d/almalinux-rt.repo
      SHELL
    end
  end
end