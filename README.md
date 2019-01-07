# Vagrant file to deploy Ceph on OpenShift

## vagrant + libvirt hostconfig

Install vagrant + vagrant-libvirt:

```
sudo yum install -y qemu libvirt libvirt-devel ruby-devel gcc qemu-kvm python-pip
sudo rpm -i https://releases.hashicorp.com/vagrant/2.0.1/vagrant_2.0.1_x86_64.rpm
vagrant plugin install vagrant-libvirt
```

Make sure your vagrant-libvirt net interface as the same content (will do the dns resolution):

```
sudo virsh net-dumpxml vagrant-libvirt
<network connections='3' ipv6='yes'>
  <name>vagrant-libvirt</name>
  <uuid>becf31bb-45c8-401e-b68a-cba0962dad3a</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:54:00:f1:e1:c7'/>
  <domain name='example.com' localOnly='yes'/>
  <ip address='192.168.121.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.121.1' end='192.168.121.254'/>
    </dhcp>
  </ip>
</network>
```


The line to add is:
`<domain name='example.com' localOnly='yes'/>`

Then:

```
sudo virsh net-edit vagrant-libvirt
sudo virsh net-destroy vagrant-libvirt
sudo virsh net-start vagrant-libvirt
sudo systemctl restart libvirtd
```

## Vagrant up!

Close OS repo:

```shell
git clone https://github.com/openshift/openshift-ansible/
cd openshift-ansible
git checkout release-3.11
git clone https://github.com/leseb/oc-ceph
mv oc-ceph/* .
pip install -r requirements.txt
```

Simply run:

`vagrant up`

Note that `vagrant provision --provision-with` can be useful to run only a subset of sequence:

* prereq-oc -> node pre-requisite to run OpenShift
* deploy-oc -> deploy OpenShift
* deploy-ceph - pending upstream

Wait wait wait until you see:

```text
PLAY RECAP *********************************************************************
k8s-master.example.com     : ok=597  changed=259  unreachable=0    failed=0
k8s-node0.example.com      : ok=185  changed=69   unreachable=0    failed=0
k8s-node1.example.com      : ok=179  changed=65   unreachable=0    failed=0
localhost                  : ok=12   changed=0    unreachable=0    failed=0


INSTALLER STATUS ***************************************************************
Initialization             : Complete
Health Check               : Complete
etcd Install               : Complete
Master Install             : Complete
Master Additional Install  : Complete
Node Install               : Complete
Hosted Install             : Complete
Service Catalog Install    : Complete

Friday 01 December 2017  09:49:23 -0500 (0:00:00.032)       0:39:22.960 *******
===============================================================================
openshift_hosted : Ensure OpenShift pod correctly rolls out (best-effort today) - 612.60s
openshift_hosted : Ensure OpenShift pod correctly rolls out (best-effort today) - 612.07s
template_service_broker : Verify that TSB is running ------------------- 89.55s
openshift_service_catalog : wait for api server to be ready ------------ 77.99s
openshift_node : Install Node package ---------------------------------- 50.43s
Run health checks (install) - EL --------------------------------------- 35.54s
openshift_ca : Install the base package for admin tooling -------------- 26.92s
openshift_cli : Install clients ---------------------------------------- 20.80s
openshift_service_catalog : oc_process --------------------------------- 18.72s
openshift_node : Install Ceph storage plugin dependencies -------------- 17.21s
restart master api ----------------------------------------------------- 13.63s
openshift_node : Install sdn-ovs package ------------------------------- 12.92s
openshift_hosted_facts : Set hosted facts ------------------------------ 12.72s
openshift_hosted_facts : Set hosted facts ------------------------------ 12.12s
openshift_hosted_facts : Set hosted facts ------------------------------ 11.56s
openshift_master : Start and enable master api on first master --------- 11.24s
openshift_manageiq : Configure role/user permissions ------------------- 10.37s
openshift_hosted_facts : Set hosted facts ------------------------------ 10.27s
os_firewall : need to pause here, otherwise the iptables service starting can sometimes cause ssh to fail -- 10.14s
os_firewall : Wait 10 seconds after disabling firewalld ---------------- 10.13s
```

## Run this on the master once the deployment is done

Log onto the master machine:

```
vagrant ssh k8s-master.example.com
sudo -i
oc get nodes
NAME                     STATUS    AGE       VERSION
k8s-master.example.com   Ready     2h        v1.7.6+a08f5eeb62
k8s-node0.example.com    Ready     2h        v1.7.6+a08f5eeb62
k8s-node1.example.com    Ready     2h        v1.7.6+a08f5eeb62
oc login -u system:admin
oadm manage-node k8s-master.example.com --schedulable=true
oadm policy add-role-to-user cluster-admin admin
```