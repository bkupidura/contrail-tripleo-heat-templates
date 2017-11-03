# Introduction    
Currently the following combinations of RHEL/OSP/Contrail are supported:    
RHEL7.4/OSP11/Contrail 4.0.2    
RHEL7.4/OSP10/Contrail 4.0.2    
RHEL7.4/OSP10/Contrail 3.2.6    
The infrastructure section should only be used as an EXAMPLE. It is not    
considered as part of OSP/Contrail deployment.    

# Infrastructure considerations
There are many different ways on how to create the infrastructure providing    
the control plane elements. In this example all control plane functions    
are provided as Virtual Machines hosted on KVM hosts. For HA 12 VMs are needed:       

- KVM 1:    
  OpenStack Controller 1   
  Contrail Controller 1    
  Contrail Analytics 1    
  Contrail Analytics DB 1    

- KVM 2:    
  OpenStack Controller 2   
  Contrail Controller 2    
  Contrail Analytics 2    
  Contrail Analytics DB 2       

- KVM 3:    
  OpenStack Controller 3   
  Contrail Controller 3    
  Contrail Analytics 3    
  Contrail Analytics DB 3    

The shown architecture is JUST an example to illustrate a possible option    
for the control plane setup.    

```
    +------------------------------------------------+    
    |                                                |    
    |  KVM host 3                                    |    
  +-----------------------------------------------+  |    
  |                                               |  |    
  |  KVM host 2                                   |  |    
+----------------------------------------------+  |  |    
|                                              |  |  |    
|  KVM host 1                                  |  |  |    
|                                              |  |  |    
|              +----------------------------+  |  |  |    
|              |                            |  |  |  |    
|              |  Contrail Analytics DB 1   |  |  |  |    
|            +---------------------------+  |  |  |  |    
|            |                           |  |  |  |  |    
|            |  Contrail Analytics 1     |  |  |  |  |    
|          +--------------------------+  |  |  |  |  |    
|          |                          |  |  |  |  |  |    
|          |  Contrail Controller 1   |  |  |  |  |  |    
|        +-------------------------+  |  +--+  |  |  |    
|        |                         |  |  |     |  |  |    
|        |  OpenStack Controller 1 |  +--+     |  |  |    
|        |                         |  |        |  |  |    
|        | +-----+ +-----+ +-----+ +--+        |  |  |    
|        | |VNIC1| |VNIC2| |VNIC3| |           |  |  |    
|        +-------------------------+           |  |  |    
|           |         |         |              |  |  |    
|           |         |         |              |  |  |    
|  +---------+  +-----------+  +---------+     |  |  |    
|  | br_prov |  | br_intapi |  | br_mgmt |     |  |  |    
|  +---------+  +-----------+  +---------+     |  +--+    
|      |             |             |           |  |    
|    +----+        +----+        +----+        +--+    
|    |NIC1|        |NIC2|        |NIC3|        |    
+----------------------------------------------+    
```

## Control plane KVM host preparation (KVM 1-3)

The control plane KVM hosts will host the control plane VMs. Each KVM host    
will need virtual switches and the virtual machine definitions. The tasks    
described must be done on each of the three hosts.    
NIC 1 - 3 have to be substituded with real NIC names.    

### Install basic packages
```
yum install -y libguestfs libguestfs-tools openvswitch virt-install kvm libvirt libvirt-python python-virtinst
```

### Start libvirtd & ovs
```
systemctl start libvirtd
systemctl start openvswitch
```

### Create virtual switches for the undercloud VM
```
ovs-vsctl add-br br_prov
ovs-vsctl add-br br_intapi
ovs-vsctl add-br br_mgmt
ovs-vsctl add-port br_prov NIC1
ovs-vsctl add-port br_intapi NIC2
ovs-vsctl add-port br_mgmt NIC3
cat << EOF > br_prov.xml
<network>
  <name>br_prov</name>
  <forward mode='bridge'/>
  <bridge name='br_prov'/>
  <virtualport type='openvswitch'/>
</network>
EOF
cat << EOF > br_intapi.xml
<network>
  <name>br_intapi</name>
  <forward mode='bridge'/>
  <bridge name='br_intapi'/>
  <virtualport type='openvswitch'/>
</network>
EOF
cat << EOF > br_mgmt.xml
<network>
  <name>br_mgmt</name>
  <forward mode='bridge'/>
  <bridge name='br_mgmt'/>
  <virtualport type='openvswitch'/>
</network>
EOF
virsh net-define br_prov.xml
virsh net-start br_prov
virsh net-autostart br_prov
virsh net-define br_intapi.xml
virsh net-start br_intapi
virsh net-autostart br_intapi
virsh net-define br_mgmt.xml
virsh net-start br_mgmt
virsh net-autostart br_mgmt
```

### Define virtual machine templates
As described above, each KVM host needs at least 4 virtual machine templates.    
For lab testing the computes can be virtualized as well, with the usual    
restrictions coming with nested HV.    

```
num=0
for i in compute control contrail-controller contrail-analytics contrail-analytics-database
do
  num=$(expr $num + 1)
  qemu-img create -f qcow2 /var/lib/libvirt/images/${i}_${num}.qcow2 40G
  virsh define /dev/stdin <<EOF
$(virt-install --name ${i}_$num --disk /var/lib/libvirt/images/${i}_${num}.qcow2 --vcpus=4 --ram=16348 --network network=br_prov,model=virtio --network network=br_intapi,model=virtio --network network=br_mgmt,model=virtio --virt-type kvm --import --os-variant rhel7 --serial pty --console pty,target_type=virtio --print-xml)
EOF
done
```

### Get provisioning interface mac addresses for ironic PXE
The virtual machines must be imported into ironic. There are different ways    
to do that. One way is to create a list of all VMs in the following format:    
MAC NODE_NAME IPMI/KVM_IP ROLE_NAME    
52:54:00:16:54:d8 control-1-at-5b3s30 10.87.64.31 control    
In order to get the initial list per KVM host the following command can be run:    
```
for i in compute control contrail-controller contrail-analytics contrail-analytics-database    
do
  prov_mac=`virsh domiflist ${i}|grep br_prov|awk '{print $5}'`
  echo ${prov_mac} ${i} >> ironic_list
done
```
The ironic_list file will contain MAC ROLE_NAME and must be manually extended    
to MAC NODE_NAME IPMI/KVM_IP ROLE_NAME.    
This is an example of a full list across three KVM hosts:    
```
52:54:00:16:54:d8 control-1-at-5b3s30 10.87.64.31 control
52:54:00:2a:7d:99 compute-1-at-5b3s30 10.87.64.31 compute
52:54:00:e0:54:b3 tsn-1-at-5b3s30 10.87.64.31 contrail-tsn
52:54:00:d6:2b:03 contrail-controller-1-at-5b3s30 10.87.64.31 contrail-controller
52:54:00:01:c1:af contrail-analytics-1-at-5b3s30 10.87.64.31 contrail-analytics
52:54:00:4a:9e:52 contrail-analytics-database-1-at-5b3s30 10.87.64.31 contrail-analytics-database
52:54:00:40:9e:13 control-1-at-centos 10.87.64.32 control
52:54:00:1d:58:4d compute-dpdk-1-at-centos 10.87.64.32 compute-dpdk
52:54:00:6d:89:2d compute-2-at-centos 10.87.64.32 compute
52:54:00:a8:46:5a contrail-controller-1-at-centos 10.87.64.32 contrail-controller
52:54:00:b3:2f:7d contrail-analytics-1-at-centos 10.87.64.32 contrail-analytics
52:54:00:59:e3:10 contrail-analytics-database-1-at-centos 10.87.64.32 contrail-analytics-database
52:54:00:1d:8c:39 control-1-at-5b3s32 10.87.64.33 control
52:54:00:9c:4b:bf compute-1-at-5b3s32 10.87.64.33 compute
52:54:00:1d:a9:d9 compute-2-at-5b3s32 10.87.64.33 compute
52:54:00:cd:59:92 contrail-controller-1-at-5b3s32 10.87.64.33 contrail-controller
52:54:00:2f:81:1a contrail-analytics-1-at-5b3s32 10.87.64.33 contrail-analytics
52:54:00:a1:4a:23 contrail-analytics-database-1-at-5b3s32 10.87.64.33 contrail-analytics-database
```

This list will be needed on the undercloud VM later on.    
With that the control plane VM KVM host preparation is done.    

## Undercloud preparation on the KVM host hosting the undercloud VM

The undercloud VM can be installed on one of the three KVM hosts or on a    
different one.    

### Set password & subscription information
```
export USER=<YOUR_RHEL_SUBS_USER>
export PASSWORD=<YOUR_RHEL_SUBS_PWD>
export POOLID=<YOUR_RHEL_POOL_ID>
export ROOTPASSWORD=<UNDERCLOUD_ROOT_PWD> # choose a root user password
export STACKPASSWORD=<STACK_USER_PWD> # choose a stack user password
```

### Install basic packages
```
yum install -y libguestfs libguestfs-tools openvswitch virt-install kvm libvirt libvirt-python python-virtinst
```

### Start libvirtd & ovs
```
systemctl start libvirtd
systemctl start openvswitch
```

### Create and become stack user
```
useradd -G libvirt stack
echo $STACKPASSWORD |passwd stack --stdin
echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
chmod 0440 /etc/sudoers.d/stack
```
### Create ssh key
```
ssh-keygen -t dsa
```

### Adjust permissions
```
chgrp -R libvirt /var/lib/libvirt/images
chmod g+rw /var/lib/libvirt/images
```

### Get rhel 7.4 kvm image
goto: https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.4/x86_64/product-software
(at the time of this writing: rhel-server-7.4-x86_64-kvm.qcow2)
download: KVM Guest Image

### Create virtual switches for the undercloud VM
```
ovs-vsctl add-br br_prov
ovs-vsctl add-br br_intapi
ovs-vsctl add-br br_mgmt
cat << EOF > br_prov.xml
<network>
  <name>br_prov</name>
  <forward mode='bridge'/>
  <bridge name='br_prov'/>
  <virtualport type='openvswitch'/>
</network>
EOF
cat << EOF > br_intapi.xml
<network>
  <name>br_intapi</name>
  <forward mode='bridge'/>
  <bridge name='br_intapi'/>
  <virtualport type='openvswitch'/>
</network>
EOF
cat << EOF > br_mgmt.xml
<network>
  <name>br_mgmt</name>
  <forward mode='bridge'/>
  <bridge name='br_mgmt'/>
  <virtualport type='openvswitch'/>
</network>
EOF
virsh net-define br_prov.xml
virsh net-start br_prov
virsh net-autostart br_prov
virsh net-define br_intapi.xml
virsh net-start br_intapi
virsh net-autostart br_intapi
virsh net-define br_mgmt.xml
virsh net-start br_mgmt
virsh net-autostart br_mgmt
```

### Prepare undercloud VM
#### OSP10
```
export LIBGUESTFS_BACKEND=direct
qemu-img create -f qcow2 undercloud.qcow2 100G
virt-resize --expand /dev/sda1 rhel-server-7.4-x86_64-kvm.qcow2 undercloud.qcow2
virt-customize  -a undercloud.qcow2 \
  --run-command 'xfs_growfs /' \
  --root-password password:$ROOTPASSWORD \
  --hostname undercloud.local \
  --sm-credentials $USER:password:$PASSWORD --sm-register --sm-attach auto --sm-attach pool:$POOLID \
  --run-command 'useradd stack' \
  --password stack:password:$STACKPASSWORD \
  --run-command 'echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack' \
  --chmod 0440:/etc/sudoers.d/stack \
  --run-command 'subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-10-rpms' \
  --install python-tripleoclient \
  --run-command 'sed -i "s/PasswordAuthentication no/PasswordAuthentication yes/g" /etc/ssh/sshd_config' \
  --run-command 'systemctl enable sshd' \
  --run-command 'yum remove -y cloud-init' \
  --selinux-relabel
cp undercloud.qcow2 /var/lib/libvirt/images/undercloud.qcow2
```

#### OSP11
```
export LIBGUESTFS_BACKEND=direct
qemu-img create -f qcow2 undercloud.qcow2 100G
virt-resize --expand /dev/sda1 rhel-server-7.4-x86_64-kvm.qcow2 undercloud.qcow2
virt-customize  -a undercloud.qcow2 \
  --run-command 'xfs_growfs /' \
  --root-password password:$ROOTPASSWORD \
  --hostname undercloud.local \
  --sm-credentials $USER:password:$PASSWORD --sm-register --sm-attach auto --sm-attach pool:$POOLID \
  --run-command 'useradd stack' \
  --password stack:password:$STACKPASSWORD \
  --run-command 'echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack' \
  --chmod 0440:/etc/sudoers.d/stack \
  --run-command 'subscription-manager repos --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms --enable=rhel-ha-for-rhel-7-server-rpms --enable=rhel-7-server-openstack-11-rpms' \
  --install python-tripleoclient \
  --run-command 'sed -i "s/PasswordAuthentication no/PasswordAuthentication yes/g" /etc/ssh/sshd_config' \
  --run-command 'systemctl enable sshd' \
  --run-command 'yum remove -y cloud-init' \
  --selinux-relabel
cp undercloud.qcow2 /var/lib/libvirt/images/undercloud.qcow2
```

### Install undercloud VM
```
virt-install --name undercloud \
  --disk /var/lib/libvirt/images/undercloud.qcow2 \
  --vcpus=4 \
  --ram=16348 \
  --network network=default,model=virtio \
  --network network=br_prov,model=virtio \
  --network network=br_intapi,model=virtio \
  --virt-type kvm \
  --import \
  --os-variant rhel7 \
  --graphics vnc \
  --serial pty \
  --noautoconsole \
  --console pty,target_type=virtio
```

### Get undercloud ip
```
virsh domifaddr undercloud
```

### Ssh into undercloud
```
ssh stack@<UNDERCLOUD_IP>
```

# Undercloud configuration

## Configure undercloud (optionally)
```
cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf
vi ~/undercloud.conf
```

## Install undercloud openstack
```
openstack undercloud install
```

## Source undercloud credentials
```
source ~/stackrc
```

## Get overcloud images
```
sudo yum install rhosp-director-images rhosp-director-images-ipa
mkdir ~/images
cd ~/images
```

## Upload overcloud images
### OSP10
```
for i in /usr/share/rhosp-director-images/overcloud-full-latest-10.0.tar /usr/share/rhosp-director-images/ironic-python-agent-latest-10.0.tar; do tar -xvf $i; done
openstack overcloud image upload --image-path /home/stack/images/
cd ~
```
### OSP11
```
for i in /usr/share/rhosp-director-images/overcloud-full-latest-11.0.tar /usr/share/rhosp-director-images/ironic-python-agent-latest-11.0.tar; do tar -xvf $i; done
openstack overcloud image upload --image-path /home/stack/images/
cd ~

## Create contrail repo
```
sudo mkdir /var/www/html/contrail
```

## Get contrail
go to:    
https://www.juniper.net/support/downloads/?p=contrail#sw

### Contrail 3.2.6
and download the 3.2.6 release (Redhat 7.3 + newton)    
transfer it to the undercloud    
```
sudo tar zxvf ~/contrail-install-packages-3.2.6.0-60-redhat73newton.tgz -C /var/www/html/contrail/
```
### Contrail 4.0.2
and download the 4.0.2 release (Redhat 7 + Contrail Networking - RHOSP10)
transfer it to the undercloud    
```
sudo tar zxvf ~/contrail-install-packages_4.0.2.0-35-newton_redhat7.tgz -C /var/www/html/contrail/
```

## Import ironic nodes
In this step the fully populated ironic_list from    
infrastructure considerations/control plane KVM host preparation (KVM 1-3)/define virtual machine templates    
is needed.    
### Virtual Machines
This imports all virtual machines into ironic.    
```
ssh_user=SSH_USER
ssh_password=SSH_PASSWORD
while IFS= read -r line
do   
  mac=`echo $line|awk '{print $1}'`
  name=`echo $line|awk '{print $2}'`
  kvm_ip=`echo $line|awk '{print $3}'`
  profile=`echo $line|awk '{print $4}'`
  uuid=`ironic node-create -d pxe_ssh -p cpus=4 -p memory_mb=16348 -p local_gb=100 -p cpu_arch=x86_64 -i ssh_username=${ssh_user} -i ssh_virt_type=virsh -i ssh_address=${kvm_ip} -i ssh_password=${ssh_password} -n $name -p capabilities=profile:${profile} | tail -2|awk '{print $4}'`
  ironic port-create -a ${mac} -n ${uuid}
done < <(cat ironic_list)
```
### Physical Machines
For importing physical compute nodes the pxe_ssh driver must be replaced with    
the ipmi driver. Easiest way is to create a ironic_list_bms with only    
physical machines in it.    
```
ipmi_user=IPMI_USER
ipmi_password=IPMI_PASSWORD
while IFS= read -r line
do
  mac=`echo $line|awk '{print $1}'`
  name=`echo $line|awk '{print $2}'`
  ipmi_address=`echo $line|awk '{print $3}'`
  profile=`echo $line|awk '{print $4}'`
  uuid=`ironic node-create -d pxe_ipmitool -p cpus=4 -p memory_mb=16348 -p local_gb=100 -p cpu_arch=x86_64 -i ipmi_username=${ipmi_user} -i ipmi_address=${ipmi_ip} -i ipmi_password=${ipmi_password} -n $name -p capabilities=profile:${profile} | tail -2|awk '{print $4}'`
  ironic port-create -a ${mac} -n ${uuid}
done < <(cat ironic_list_bms)
```

## Configure boot mode
```
openstack baremetal configure boot
```

## Node introspection
```
for node in $(openstack baremetal node list -c UUID -f value) ; do openstack baremetal node manage $node ; done
openstack overcloud node introspect --all-manageable --provide
```

## Node profiling
```
for i in contrail-controller contrail-analytics contrail-database contrail-analytics-database; do
  openstack flavor create $i --ram 4096 --vcpus 1 --disk 40
  openstack flavor set --property "capabilities:boot_option"="local" --property "capabilities:profile"="${i}" ${i}
done
```

# Configure overcloud

## Install tripleo-heat-templates on the undercloud
### Contrail 3.2.6
```
yum localinstall /var/www/html/contrail/contrail-tripleo-heat-templates-3.2.6.0-60.el7.noarch.rpm
```
### Contrail 4.0.2
```
yum localinstall /var/www/html/contrail/contrail-tripleo-heat-templates-4.0.2.0-35.el7.noarch.rpm
```
```
cp -r /usr/share/openstack-tripleo-heat-templates/ ~/tripleo-heat-templates
cp -r contrail-tripleo-heat-templates/environments/* ~/tripleo-heat-templates/environments
cp -r contrail-tripleo-heat-templates/puppet/services/network/* ~/tripleo-heat-templates/puppet/services/network
```

## Contrail services (repo url etc.)
### Set Contrail version
#### Contrail 3.2.6
set ContrailVersion: 3 in ~/tripleo-heat-templates/environments/contrail/contrail-services.yaml    
```
vi ~/tripleo-heat-templates/environments/contrail/contrail-services.yaml
```
#### Contrail 4.0.2
4.0.2 is default    

## Overcloud networking
### NIC configurations
#### OSP10
```
vi ~/tripleo-heat-templates/environments/contrail/contrail-net.yaml
vi ~/tripleo-heat-templates/environments/contrail/contrail-nic-config-compute.yaml
vi ~/tripleo-heat-templates/environments/contrail/contrail-nic-config.yaml
```
#### OSP11
```
vi ~/tripleo-heat-templates/environments/contrail/contrail-net.yaml
vi ~/tripleo-heat-templates/environments/configs/contrail/contrail-nic-config-compute.yaml
vi ~/tripleo-heat-templates/environments/configs/contrail/contrail-nic-config.yaml
```

### Static ip assignment
#### OSP10
```
vi ~/tripleo-heat-templates/environments/contrail/ips-from-pool-all.yaml
```
#### OSP11
```
vi ~/tripleo-heat-templates/environments/ips-from-pool-all.yaml
```

## Provide subscription mgr credentials (rhel_reg_password, rhel_reg_pool_id, rhel_reg_repos, rhel_reg_user and method)
```
vi ~/tripleo-heat-templates/extraconfig/pre_deploy/rhel-registration/environment-rhel-registration.yaml
```

# Start overcloud installation
## Contrail 3.2.6
```
openstack overcloud deploy --templates tripleo-heat-templates/ \
  --roles-file tripleo-heat-templates/environments/contrail/roles_data.yaml \
  -e tripleo-heat-templates/environments/puppet-pacemaker.yaml \
  -e tripleo-heat-templates/environments/contrail/contrail-services.yaml \
  -e tripleo-heat-templates/environments/contrail/network-isolation.yaml \
  -e tripleo-heat-templates/environments/contrail/contrail-net.yaml \
  -e tripleo-heat-templates/environments/contrail/ips-from-pool-all.yaml \
  -e tripleo-heat-templates/environments/network-management.yaml \
  -e tripleo-heat-templates/extraconfig/pre_deploy/rhel-registration/environment-rhel-registration.yaml \
  -e tripleo-heat-templates/extraconfig/pre_deploy/rhel-registration/rhel-registration-resource-registry.yaml \
  --libvirt-type qemu
```

## Contrail 4.0.2
```
openstack overcloud deploy --templates tripleo-heat-templates/ \
  --roles-file tripleo-heat-templates/environments/contrail/roles_data.yaml \
  -e tripleo-heat-templates/environments/puppet-pacemaker.yaml \
  -e tripleo-heat-templates/environments/contrail/contrail-services.yaml \
  -e tripleo-heat-templates/environments/network-isolation.yaml \
  -e tripleo-heat-templates/environments/contrail/contrail-net.yaml \
  -e tripleo-heat-templates/environments/ips-from-pool-all.yaml \
  -e tripleo-heat-templates/environments/network-management.yaml \
  -e tripleo-heat-templates/extraconfig/pre_deploy/rhel-registration/environment-rhel-registration.yaml \
  -e tripleo-heat-templates/extraconfig/pre_deploy/rhel-registration/rhel-registration-resource-registry.yaml \
  --libvirt-type qemu
```
