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
        c = (0..NNODES - 1).map { |j| "#{LABEL_PREFIX}node#{j}.example.com" }
        n[NNODES] = "#{LABEL_PREFIX}master.example.com"
        NNODES = NNODES + 1

        # first run - node preparation
        node.vm.provision "node-prep", type: "ansible" do |ansible|
          ansible.playbook = 'node-prep.yml'
          # decent verbosity
          ansible.verbose = '-vv'
          if DEBUG then
            ansible.verbose = '-vvvv'
          end
          # the openshift-ansible playbook does not support limit somehow, so it'll skip all the nodes
          ansible.limit = ""
        end

        # second run - openshift pre-req
        node.vm.provision "prereq-oc", type: "ansible" do |ansible|
          ansible.playbook = 'playbooks/prerequisites.yml'
          ansible.host_vars = {
            "k8s-master.example.com" => "openshift_node_group_name='node-config-master'",
            "k8s-node0.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node1.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node2.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node3.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node4.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node5.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node6.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node7.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node8.example.com"  => "openshift_node_group_name='node-config-compute'",
          }
          ansible.groups = {
            "nodes"           => n,
            "ceph"            => c,
            "masters"         => ["#{LABEL_PREFIX}master.example.com"],
            "etcd"            => ["#{LABEL_PREFIX}master.example.com"],
            "OSEv3:children"  => ["masters", "nodes", "etcd", "ceph"],
            "OSEv3:vars"      => {
              "deployment_type" => "origin",
              "openshift_deployment_type" => "origin",
              "ansible_become" => true,
              "openshift_repos_enable_testing" => true,
              "openshift_enable_excluders" => false,
              "enable_docker_excluder" => false,
              "openshift_cluster_monitoring_operator_install" => false, # we disable this because we don't have any 'infra' node
              "openshift_disable_check" => "memory_availability,docker_storage,disk_availability",
              "openshift_additional_repos" => "[{'id': 'centos-paas', 'name': 'centos-paas', 'baseurl' :'https://buildlogs.centos.org/centos/7/paas/x86_64/openshift-origin311', 'gpgcheck' :'0', 'enabled' :'1'}]",
            },
          }
          # decent verbosity
          ansible.verbose = '-vv'
          if DEBUG then
            ansible.verbose = '-vvvv'
          end
          # the openshift-ansible playbook does not support limit somehow, so it'll skip all the nodes
          ansible.limit = ""
        end

        # third run - deploy openshift
        node.vm.provision "deploy-oc", type: "ansible" do |ansible|
          ansible.playbook = 'playbooks/deploy_cluster.yml'
          ansible.host_vars = {
            "k8s-master.example.com" => "openshift_node_group_name='node-config-master'",
            "k8s-node0.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node1.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node2.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node3.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node4.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node5.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node6.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node7.example.com"  => "openshift_node_group_name='node-config-compute'",
            "k8s-node8.example.com"  => "openshift_node_group_name='node-config-compute'",
          }
          ansible.groups = {
            "nodes"           => n,
            "ceph"            => c,
            "masters"         => ["#{LABEL_PREFIX}master.example.com"],
            "etcd"            => ["#{LABEL_PREFIX}master.example.com"],
            "OSEv3:children"  => ["masters", "nodes", "etcd", "ceph"],
            "OSEv3:vars"      => {
              "deployment_type" => "origin",
              "openshift_deployment_type" => "origin",
              "ansible_become" => true,
              "openshift_repos_enable_testing" => true,
              "openshift_enable_excluders" => false,
              "enable_docker_excluder" => false,
              "openshift_cluster_monitoring_operator_install" => false, # we disable this because we don't have any 'infra' node
              "openshift_disable_check" => "memory_availability,docker_storage,disk_availability",
              "openshift_additional_repos" => "[{'id': 'centos-paas', 'name': 'centos-paas', 'baseurl' :'https://buildlogs.centos.org/centos/7/paas/x86_64/openshift-origin311', 'gpgcheck' :'0', 'enabled' :'1'}]",
              "openshift_storage_ceph_install" => true,
            },
          }
          # decent verbosity
          ansible.verbose = '-vv'
          if DEBUG then
            ansible.verbose = '-vvvv'
          end
          # the openshift-ansible playbook does not support limit somehow, so it'll skip all the nodes
          ansible.limit = ""
        end

        # forth run - deploy ceph
        # node.vm.provision "deploy-ceph", type: "ansible" do |ansible|
        #   ansible.playbook = 'playbooks/openshift-ceph/config.yml'
        #   ansible.host_vars = {
        #     "k8s-master.example.com" => "openshift_node_group_name='node-config-master'",
        #     "k8s-node0.example.com"  => "openshift_node_group_name='node-config-compute'",
        #     "k8s-node1.example.com"  => "openshift_node_group_name='node-config-compute'",
        #     "k8s-node2.example.com"  => "openshift_node_group_name='node-config-compute'",
        #     "k8s-node3.example.com"  => "openshift_node_group_name='node-config-compute'",
        #     "k8s-node4.example.com"  => "openshift_node_group_name='node-config-compute'",
        #     "k8s-node5.example.com"  => "openshift_node_group_name='node-config-compute'",
        #     "k8s-node6.example.com"  => "openshift_node_group_name='node-config-compute'",
        #     "k8s-node7.example.com"  => "openshift_node_group_name='node-config-compute'",
        #     "k8s-node8.example.com"  => "openshift_node_group_name='node-config-compute'",
        #   }
        #   ansible.groups = {
        #     "nodes"           => n,
        #     "ceph"            => c,
        #     "masters"         => ["#{LABEL_PREFIX}master.example.com"],
        #     "etcd"            => ["#{LABEL_PREFIX}master.example.com"],
        #     "OSEv3:children"  => ["masters", "nodes", "etcd", "ceph"],
        #     "OSEv3:vars"      => {
        #       "deployment_type" => "origin",
        #       "openshift_deployment_type" => "origin",
        #       "ansible_become" => true,
        #       "openshift_repos_enable_testing" => true,
        #       "openshift_enable_excluders" => false,
        #       "enable_docker_excluder" => false,
        #       "openshift_cluster_monitoring_operator_install" => false, # we disable this because we don't have any 'infra' node
        #       "openshift_disable_check" => "memory_availability,docker_storage,disk_availability",
        #       "openshift_additional_repos" => "[{'id': 'centos-paas', 'name': 'centos-paas', 'baseurl' :'https://buildlogs.centos.org/centos/7/paas/x86_64/openshift-origin311', 'gpgcheck' :'0', 'enabled' :'1'}]",
        #     },
        #   }
        #   # decent verbosity
        #   ansible.verbose = '-vv'
        #   if DEBUG then
        #     ansible.verbose = '-vvvv'
        #   end
        #   # the openshift-ansible playbook does not support limit somehow, so it'll skip all the nodes
        #   ansible.limit = ""
        # end

      end
    end
  end
end
