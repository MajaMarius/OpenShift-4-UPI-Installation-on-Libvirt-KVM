# OpenShift-4-UPI-Installation-on-Libvirt-KVM

sudo su - 

yum update

yum install -y qemu-kvm qemu-img virt-manager libvirt libvirt-python libvirt-client virt-install virt-viewer bridge-utils libguestfs-tools wget vim


systemctl start libvirtd

systemctl enable libvirtd


Create a new directory named for example ocp4 and go in it

    mkdir ocp4
    cd ocp4

### Make sure that you have "default"  libvirt’s network running  

    virsh net-list
    

```

VIR_NET="default"
HOST_NET=$(ip -4 a s $(virsh net-info $VIR_NET | awk '/Bridge:/{print $2}') | awk '/inet /{print $2}')
HOST_IP=$(echo $HOST_NET | cut -d '/' -f1)
#DNS_DIR="/etc/NetworkManager/dnsmasq.d"
BASE_DOM="test"
WEB_PORT='1234'
CLUSTER_NAME="mylab"
SSH_KEY=<ssh pub key>
PULL_SEC='<paste-pull-secret>'
 
```
  
  Download your pull secret from Red Hat OpenShift Cluster Manager and load into a variable. You can also copy paste the pull secret. The variable PULL_SEC should have your pull secret without any newlines.
  SSH key-ed25519 can be generated with following commands:
  
    ssh-keygen -t d25519
   

  Create a folder "rhcos-install"
  Download the RHCOS kernel and initramfs images

```
mkdir rhcos-install
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/4.5.6/rhcos-4.5.6-x86_64-installer-kernel-x86_64 -O ./rhcos-install/vmlinuz
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/4.5.6/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img -O ./rhcos-install/initramfs.img
```

Generate treeinfo

```
cat <<EOF > rhcos-install/.treeinfo
[general]
arch = x86_64
family = Red Hat CoreOS
platforms = x86_64
version = 4.5.6
[images-x86_64]
initrd = initramfs.img
kernel = vmlinuz
EOF
```

  Download the RHCOS bios image
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/4.5.6/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz

Create folder for Vms disks and download Centos image
    
    mkdir -p /data/VMs
    wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2 -O /data/VMs/${CLUSTER_NAME}-lb.qcow2

Create LB machine disk


```
virt-customize -a  /data/VMs/${CLUSTER_NAME}-lb.qcow2 \
  --root-password password:redhat \
  --uninstall cloud-init \
  --ssh-inject root:file:$SSH_KEY --selinux-relabel
```

Create the machine

```
virt-install --import --name lb.${CLUSTER_NAME}.test \
  --disk /var/lib/libvirt/images/${CLUSTER_NAME}-lb.qcow2,size=80 --memory 2048 --cpu host --vcpus 4 \
  --network network=${VIR_NET},mac=52:54:00:aa:04:00 --noreboot --noautoconsole \
  --graphics vnc,listen=0.0.0.0
```

Enter in LB Vm
virsh console <Lbname>

Download OpenShift client and install binaries and extract 

```
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.5.6/openshift-install-linux-4.5.6.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.5.6/openshift-client-linux-4.5.6.tar.gz

tar xf openshift-install-linux-4.5.6.tar.gz
tar xf openshift-client-linux-4.5.6.tar.gz
rm -f README.md
```

Create the installation directory for the OpenShift installer:

    mkdir install_dir
   
Generate the install-config.yaml:

```
apiVersion: v1
baseDomain: test
compute:
- hyperthreading: Disabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Disabled
  name: master
  replicas: 3
metadata:
  name: mylab
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret:
sshKey:
```

Generate the ignition file

    ./openshift-install create ignition-configs --dir=./install_dir

Check that VMs can access the host on the web port and open a HTTP server to serve the ignition file

    iptables -I INPUT -p tcp -m tcp --dport ${WEB_PORT} -s ${HOST_NET} -j ACCEPT
    python -m SimpleHTTPServer ${WEB_PORT}
    
    
 Create the VMs, Bootstrap, 3workers and 3masters
 
 
 ```
mac=1
virt-install --name bootstrap.${CLUSTER_NAME}.test \
  --disk /data/VMs/${CLUSTER_NAME}-bootstrap.qcow2,size=50 --ram 14000 --cpu host --vcpus 4 \ 
  --os-type linux --os-variant rhel7 \
  --graphics vnc,listen=0.0.0.0 \
  --network network=${VIR_NET},mac=52:54:00:aa:04:0${mac} --noautoconsole \
  --location boot/rhcos-install/ \
  --extra-args "nomodeset rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.inst.image_url=http://${HOST_IP}:${WEB_PORT}/boot/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz  coreos.inst.ignition_url=http://${HOST_IP}:${WEB_PORT}/install_dir/bootstrap.ign"
mac=$(( $mac + 1 ))
for i in {1..3}; do
  virt-install --name master-${i}.${CLUSTER_NAME}.test \
    --disk /data/VMs/${CLUSTER_NAME}-master-${i}.qcow2,size=60 --ram 16384 --cpu host --vcpus 5 \ 
    --os-type linux --os-variant rhel7 \
    --graphics vnc,listen=0.0.0.0 \
    --network network=${VIR_NET},mac=52:54:00:aa:04:0${mac} --noautoconsole \
    --location boot/rhcos-install/ \
    --extra-args "nomodeset rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.inst.image_url=http://${HOST_IP}:${WEB_PORT}/boot/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://${HOST_IP}:${WEB_PORT}/install_dir/master.ign"
    mac=$(( $mac + 1 ))
done
for i in {1..3}; do
  virt-install --name worker-${i}.${CLUSTER_NAME}.test \
    --disk /data/VMs/${CLUSTER_NAME}-worker-${i}.qcow2,size=60 --ram 16384 --cpu host --vcpus 5 \ 
    --os-type linux --os-variant rhel7 \
    --graphics vnc,listen=0.0.0.0 \
    --network network=${VIR_NET},mac=52:54:00:aa:04:0${mac} --noautoconsole \
    --location boot/rhcos-install/ \
    --extra-args "nomodeset rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.inst.image_url=http://${HOST_IP}:${WEB_PORT}/boot/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz coreos.inst.ignition_url=http://${HOST_IP}:${WEB_PORT}/install_dir/worker.ign"
    mac=$(( $mac + 1 ))
done
```
 



 
 








