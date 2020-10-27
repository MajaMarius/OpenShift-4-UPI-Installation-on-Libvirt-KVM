# OpenShift-4-UPI-Installation-on-Libvirt-KVM

 
'''

VIR_NET="default"
HOST_NET=$(ip -4 a s $(virsh net-info $VIR_NET | awk '/Bridge:/{print $2}') | awk '/inet /{print $2}')
HOST_IP=$(echo $HOST_NET | cut -d '/' -f1)
#DNS_DIR="/etc/NetworkManager/dnsmasq.d"
BASE_DOM="test"
WEB_PORT='1234'
CLUSTER_NAME="mylab"
SSH_KEY=<ssh pub key>
 
'''
  
  
  Download your pull secret from Red Hat OpenShift Cluster Manager and load into a variable. You can also copy paste the pull secret. The variable PULL_SEC should have your pull secret without any newlines.
  
PULL_SEC='<paste-pull-secret>'
  
  Download the RHCOS Install kernel and initramfs and generate the treeinfo
  
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/4.5.6/rhcos-4.5.6-x86_64-installer-kernel-x86_64 -O rhcos-install/vmlinuz
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/4.5.6/rhcos-4.5.6-x86_64-installer-initramfs.x86_64.img -O rhcos-install/initramfs.img

  Download the RHCOS bios image
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.5/4.5.6/rhcos-4.5.6-x86_64-metal.x86_64.raw.gz


