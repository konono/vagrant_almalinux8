# Install Vagrant-libvirt
### AlmaLinux 8:
```
Plz check latest version.
curl https://releases.hashicorp.com/vagrant/ |grep vagrant_

e.g
sudo dnf install https://releases.hashicorp.com/vagrant/2.2.18/vagrant_2.2.18_x86_64.rpm

```

### Install dependency packages
```
dnf install qemu-kvm qemu-img libvirt virt-install libvirt-client libvirt-devel ruby-devel gcc libxslt-devel libxml2-devel  libguestfs-tools-c ruby-devel flex bison gcc make cmake ruby-devel
```

### Prepare modules for vagrant-libvirt
```
# compile krb5
cd /tmp/; wget http://vault.centos.org/8.2.2004/BaseOS/Source/SPackages/krb5-1.17-18.el8.src.rpm
rpm2cpio krb5-1.17-18.el8.src.rpm | cpio -imdV
tar xf krb5-1.17.tar.gz
cd krb5-1.17/src
LDFLAGS='-L/opt/vagrant/embedded/' ./configure
make
sudo cp lib/libk5crypto.so.3 /opt/vagrant/embedded/lib64/

# compile libssh
cd /tmp/; dnf download --source libssh
rpm2cpio libssh-0.9.4-2.el8.src.rpm | cpio -imdV
tar xf libssh-0.9.4.tar.xz
mkdir build_libssh
cd build_libssh/
cmake ../libssh-0.9.4 -DOPENSSL_ROOT_DIR=/opt/vagrant/embedded/
make
sudo cp lib/libssh* /opt/vagrant/embedded/lib64
```

### Install vagrant-libvirt
```
CONFIGURE_ARGS='with-ldflags=-L/opt/vagrant/embedded/lib with-libvirt-include=/usr/include/libvirt with-libvirt-lib=/usr/lib' GEM_HOME=~/.vagrant.d/gems GEM_PATH=$GEM_HOME:/opt/vagrant/embedded/gems PATH=/opt/vagrant/embedded/bin:$PATH vagrant plugin install vagrant-libvirt
```

###  Post configuration if you use synced_folder. **Optional**
```
dnf -y install nfs-utils
systemctl enable --now rpcbind nfs-server
firewall-cmd --zone=libvirt --add-service=nfs
firewall-cmd --permanent --zone=libvirt --add-service=nfs
```

### Create Quickstart Vagrantfile
And then make a Vagrantfile that looks like the following, filling in your information where necessary. For example:
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

### Set some variables
# Path to the local users public key file in $HOME/.ssh
# We use it later in the shell provisioner to populate authorized_keys
ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip

  ###------------------------------   Main   -------------------------------###

Vagrant.configure("2") do |config|
  #config.ssh.username = 'vagrant'
  config.ssh.username = 'root'
  config.ssh.password = 'vagrant'
  config.ssh.insert_key = 'false'
  # In order to use this configuration, you need to install NFS. If you want to use it, please install the Post configuration above.
  # config.vm.synced_folder "./share", "/root/share",
     type: "nfs", nfs_version: "4", nfs_udp: false

  config.vm.define :sv1 do |sv1|
    sv1.vm.box = "almalinux/8"
    sv1.vm.box_check_update = false
    sv1.vm.hostname = "sv1"

    sv1.vm.network :private_network,
      ip: '10.0.0.2',
      libvirt__network_name: "net1",
      libvirt__dhcp_enabled: false,
      libvirt__host_ip: '10.0.0.1',
      libvirt__forward_mode: 'none'

    sv1.vm.provider :libvirt do |lv|
      lv.cpus = 4
      lv.memory = 8192
      lv.boot 'hd'
      lv.machine_virtual_size = 100
      lv.cpu_mode = 'host-passthrough'
      lv.machine_arch = "x86_64"
      lv.autostart = true
      lv.nested = true
      lv.uri = 'qemu+unix:///system'
      lv.management_network_name = 'default'
      lv.management_network_address = "192.168.122.0/24"
      lv.forward_ssh_port = true

    sv1.vm.network :forwarded_port, guest: 22, host: 10022, host_ip: "0.0.0.0", id: "ssh"

    end ###--- End Provider

  end ###--- End VM.Define

  ###------------------------------   Provisioning   -------------------------------###
  config.vm.provision "Setup shell environment", type: "shell" do |s|
    s.inline = <<-SHELL

    # Add the public keys "adminvm" is a VM I use for testing things like Ansible
    # Clearlinux box doesn't havre a vagrant user - the user is "clear"
    mkdir -p /root/.ssh
    chmod 700 /root/.ssh
    touch /root/.ssh/authorized_keys

    echo "Appending user@Laptop keys to root and vagrant authorized_keys"
    echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
    echo #{ssh_pub_key} >> /root/.ssh/authorized_keys

    echo "Expand root storage"

    sudo dnf install -y cloud-utils-growpart
    sudo growpart /dev/vda 2
    sudo xfs_growfs /

    SHELL

  end ###--- End Inline Provisioner

  ###------------------------------   Trigger   -------------------------------###
  config.trigger.after :up do |trigger|
    trigger.info = "Create sh config after up operation"
    trigger.run = {inline: "bash -c 'mkdir -p ~/.ssh/conf.d/hosts/ && vagrant ssh-config > ~/.ssh/conf.d/hosts/${PWD##*/}' "}
  end ###--- End Trigger

  config.trigger.after :destroy do |trigger|
    trigger.info = "Delete ssh config after destroy operation"
    trigger.run = {inline: "bash -c 'rm -rf ~/.ssh/conf.d/hosts/${PWD##*/}' "}
  end ###--- End Trigger

end #    EOF
```
### TIPS

How to change private network bridge name
```
$ vim /home/ubuntu/.vagrant.d/gems/gems/vagrant-libvirt-0.0.43/lib/vagrant-libvirt/action/create_networks.rb

#:277     while lookup_bridge_by_name(bridge_name = "virbr#{count}") <--- You can change this name to whatever you like.
```

Happy Vagrant :)
