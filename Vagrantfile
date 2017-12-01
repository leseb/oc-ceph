# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'
require 'time'
VAGRANTFILE_API_VERSION = '2'

config_file=File.expand_path(File.join(File.dirname(__FILE__), 'vagrant_variables.yml'))
settings=YAML.load_file(config_file)

LABEL_PREFIX    = settings['label_prefix'] ? settings['label_prefix'] + "-" : ""
NNODES           = settings['node_vms']
PUBLIC_SUBNET   = settings['public_subnet']
CLUSTER_SUBNET  = settings['cluster_subnet']
BOX             = settings['vagrant_box']
BOX_URL         = settings['vagrant_box_url']
MEMORY          = settings['memory']
DEBUG           = settings['debug']
DISK_UUID = Time.now.utc.to_i

system("
    if [ #{ARGV[0]} = 'up' ] || [ #{ARGV[0]} = 'provision' ]; then
	timeout 5 vagrant ssh-config > ssh-config && sed 's/vagrant/root/g' -i ssh-config
    fi
")

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = BOX
  config.vm.box_url = BOX_URL
  config.ssh.insert_key = false # workaround for https://github.com/mitchellh/vagrant/issues/5048
  config.ssh.username = 'vagrant'

  # Override
  config.vm.provider :libvirt do |v,override|
    override.vm.synced_folder '.', '/vagrant', disabled: true
  end

  # Let's speed up a little bit
  config.vm.provider :libvirt do |lv|
    lv.cpu_mode = 'host-passthrough'
    lv.volume_cache = 'unsafe'
    lv.graphics_type = 'none'
  end

  config.vm.define "#{LABEL_PREFIX}master.example.com" do |master|
    master.vm.box = BOX
    master.vm.hostname = "#{LABEL_PREFIX}master"
    master.vm.network :private_network, ip: "#{PUBLIC_SUBNET}.4"

    master.vm.provider :libvirt do |lv|
      lv.storage :file, :device => "hdb", :path => "disk-master-0-#{DISK_UUID}.disk", :size => '80G', :bus => "ide"
      lv.memory = MEMORY
      lv.random_hostname = true
    end
  end

  (0..NNODES - 1).each do |i|
    config.vm.define "#{LABEL_PREFIX}node#{i}.example.com" do |node|
      node.vm.hostname = "#{LABEL_PREFIX}node#{i}"
      node.vm.network :private_network, ip: "#{PUBLIC_SUBNET}.10#{i}"
      node.vm.network :private_network, ip: "#{CLUSTER_SUBNET}.20#{i}"

      driverletters = ('a'..'z').to_a
      node.vm.provider :libvirt do |lv|
        (0..3).each do |d|
          lv.storage :file, :device => "hd#{driverletters[d]}", :path => "disk-#{i}-#{d}-#{DISK_UUID}.disk", :size => '80G', :bus => "ide"
        end
        lv.memory = MEMORY
        lv.random_hostname = true
      end
      if i == (NNODES-1)
        n = (0..NNODES - 1).map { |j| "#{LABEL_PREFIX}node#{j}.example.com" }
    	n[NNODES] = "#{LABEL_PREFIX}master.example.com"
    	node.vm.provision :ansible do |ansible|
    	  ansible.playbook = 'playbooks/byo/config.yml'
    	  ansible.host_vars = {
  	      "k8s-master.example.com" => "openshift_node_labels=\"{'region': 'infra', 'zone': 'default'}\"",
  	      "k8s-node0.example.com" => "openshift_node_labels=\"{'region': 'primary', 'zone': 'west'}\"",
  	      "k8s-node1.example.com" => "openshift_node_labels=\"{'region': 'primary', 'zone': 'west'}\"",
  	      "k8s-node2.example.com" => "openshift_node_labels=\"{'region': 'primary', 'zone': 'west'}\"",
  	      "k8s-node3.example.com" => "openshift_node_labels=\"{'region': 'primary', 'zone': 'west'}\"",
  	      "k8s-node4.example.com" => "openshift_node_labels=\"{'region': 'primary', 'zone': 'west'}\"",
  	      "k8s-node5.example.com" => "openshift_node_labels=\"{'region': 'primary', 'zone': 'west'}\"",
  	      "k8s-node6.example.com" => "openshift_node_labels=\"{'region': 'primary', 'zone': 'west'}\"",
  	      "k8s-node7.example.com" => "openshift_node_labels=\"{'region': 'primary', 'zone': 'west'}\"",
  	      "k8s-node8.example.com" => "openshift_node_labels=\"{'region': 'primary', 'zone': 'west'}\"",
  	    }
    	  ansible.groups = {
    	    "nodes"             => n,
    	    "masters"          => ["#{LABEL_PREFIX}master.example.com"],
    	    "etcd" => ["#{LABEL_PREFIX}master.example.com"],
    	    "OSEv3:children" => ["masters", "nodes"],
    	    "OSEv3:vars" => {"deployment_type" => "origin", "ansible_ssh_user" => "root" },
  	    }
    	  ansible.extra_vars = {
  	      openshift_disable_check: "disk_availability,memory_availability",
    	  }
    	  if DEBUG then
    	    ansible.verbose = '-vvvv'
    	  end
    	  ansible.limit = 'all'
    	end
      end
    end
  end

  $script = "
yum -y update
yum -y install vim  wget git net-tools bind-utils iptables-services bridge-utils bash-completion pyOpenSSL docker
cat <<EOF > /etc/sysconfig/docker-storage-setup
DEVS=/dev/sda
VG=docker-vg
EOF
docker-storage-setup
if ! grep -Eo INSECURE_REGISTRY /etc/sysconfig/docker ; then echo \"INSECURE_REGISTRY='--selinux-enabled --insecure-registry 172.30.0.0\/16'\" | sudo tee -a /etc/sysconfig/docker; fi
systemctl enable docker
systemctl start docker
if [[ \"$(hostname -f)\" != \"$(uname -n).example.com\" ]] ; then hostnamectl set-hostname $(uname -n).example.com ; fi
if ! grep -Eo DHCP_HOSTNAME /etc/sysconfig/network-scripts/ifcfg-eth0; then echo \"DHCP_HOSTNAME=$(uname -n)\" | sudo tee -a /etc/sysconfig/network-scripts/ifcfg-eth0 ; fi
mkdir -p /root/.ssh
cp -v /home/vagrant/.ssh/authorized_keys /root/.ssh
systemctl restart network
    "
  config.vm.provision 'shell', inline: $script
end

