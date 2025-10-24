#### Phase 0 - Set Up


# Build VMs 
./setup/create_vms.ps1
# K8s node all get a second NIC in 'k8s-trunk', and a 500GB secondary hard disk 


# Passwordless Sudo
sudo visudo
user ALL=(ALL) NOPASSWD:ALL

# Install Node Prerequisites for K8s 
sudo apt install -y chrony
sudo systemctl enable --now chronyd
chronyc sources

# 6.8 Kernel
sudo apt update  -y && sudo apt upgrade -y && sudo apt install open-vm-tools -y
sudo apt install --install-recommends linux-lowlatency-hwe-22.04 -y
sudo update-grub
sudo reboot now 


# Fix netfilter otherwise flannel will error 
sudo modprobe br_netfilter
sudo modprobe overlay

sudo echo br_netfilter | sudo tee /etc/modules-load.d/k8s.conf
sudo echo overlay | sudo tee /etc/modules-load.d/k8s.conf


# Install required packages on K8s nodes  for OpenStack
sudo apt install -y python3-pip git jq curl ceph-common bridge-utils vlan  # openvswitch-switch

# Enable kernel modules for networking
sudo tee /etc/modules-load.d/openstack.conf <<EOF
br_netfilter
overlay
openvswitch
8021q
EOF

sudo modprobe br_netfilter overlay openvswitch 8021q


sudo sysctl --system

# Enable it on boot and set sysctl values:
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
# Verify the file exists:
sudo cat /proc/sys/net/bridge/bridge-nf-call-iptables
# It should print 1.


# Tweak Network - Check current limits
cat /proc/sys/fs/inotify/max_user_watches
cat /proc/sys/fs/inotify/max_user_instances

# Temporarily increase limits (until reboot)
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512

# Make it permanent across reboots
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf


# Disable swap
sudo swapoff -a
sudo vi /etc/fstab 
# Comment out the /swap line. Save and quit. 

#  Fix IP Forwarding 
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 2. Install Container Runtime (containerd)
# Kubernetes 1.30+ works best with containerd.

sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Install k9s
curl -LO https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz
tar -xvf k9s_Linux_amd64.tar.gz
sudo mv k9s /usr/local/bin/
rm LICENSE
rm README.md
rm k9s_Linux_amd64.tar.gz



# Get ready for ansible
ssh-keygen
ssh-copy-id user@k8s-n1
ssh-copy-id user@k8s-n2
ssh-copy-id user@k8s-n3


# Generic Resize Disk  Ubuntu 22
sudo growpart /dev/sda 3
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

sudo apt autoremove -y

########## Snap VMs
$snapname = 'Try 3 - Clean'
$description = get-date
get-vm 'k8s-ctl','k8s-n1','k8s-n2','k8s-n3' | new-snapshot -Name $snapname -Description $description -Quiesce -Memory
##########

# Clean up from the Helm Attempt
sudo rm /etc/sysctl.d/99-openstack.conf 
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
sudo reboot now
#

# Install required packages
sudo apt update
sudo apt install -y open-iscsi nfs-common python3.11 python3.11-venv git

# Enable and start iSCSI for Longhorn
sudo systemctl enable iscsid
sudo systemctl start iscsid

# On all nodes, enable k8s repo 
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

## Install required components
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# On k8s-ctl (control node)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.233.0.0/16 \
  --apiserver-advertise-address=192.168.1.24

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


# On worker nodes (k8s-n1, k8s-n2, k8s-n3)
# Use the join command from kubeadm init output
sudo kubeadm join 192.168.1.24:6443 --token mpnqgx.q2c0u4po18hnl14e \
        --discovery-token-ca-cert-hash sha256:962f11e25bae5ead107c1b932578c7e3d7bd70481500d1fd39857fa64bcaeb99

## Install Tigera operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/tigera-operator.yaml

# Download and customize installation
curl https://raw.githubusercontent.com/projectcalico/calico/v3.30.3/manifests/custom-resources.yaml -o calico-custom.yaml

# Edit the file to reflect the correct cidr shown below:
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16  # Kubernetes pod network
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
#

# Apply Config - Verify all running before continuing
kubectl create -f calico-custom.yaml
watch kubectl get pods -n calico-system

# Install Longhorn via Helm
sudo snap install helm -y
helm repo add longhorn https://charts.longhorn.io
helm repo update

# Create values file for production configuration
cat > longhorn-values.yaml <<EOF
defaultSettings:
  defaultReplicaCount: 3
  defaultDataPath: /var/lib/longhorn
  defaultDataLocality: disabled
  replicaSoftAntiAffinity: true
  storageMinimalAvailablePercentage: 10
  
persistence:
  defaultClass: true
  defaultClassReplicaCount: 3
  reclaimPolicy: Retain
  
ingress:
  enabled: false
EOF

## Install Longhorn
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --values longhorn-values.yaml \
  --version 1.9.1

# Wait for all pods to be ready
kubectl get pods -n longhorn-system -w

# Label storage nodes
kubectl label nodes k8s-n1 k8s-n2 k8s-n3 longhorn.io/storage-node=enabled

# Label all worker nodes to allow neutron-northd
kubectl label nodes k8s-n1 k8s-n2 k8s-n3 network.yaook.cloud/neutron-northd=true

# Add the nova-any-service label to all compute nodes
kubectl label nodes k8s-n1 k8s-n2 k8s-n3 compute.yaook.cloud/nova-any-service=true

# Verify the labels were added
kubectl get nodes --show-labels | grep nova-any-service


cat > longhorn-database-sc.yaml <<EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: longhorn-database
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
volumeBindingMode: Immediate
reclaimPolicy: Retain
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  dataLocality: "best-effort"
  fsType: "ext4"
EOF

kubectl apply -f longhorn-database-sc.yaml


# Configure longhorn disk sdb
sudo mkfs.ext4 -F /dev/sdb
sudo mkdir -p /var/lib/longhorn/sdb
sudo mount /dev/sdb /var/lib/longhorn/sdb
echo "/dev/sdb /var/lib/longhorn/sdb ext4 defaults 0 0" | sudo tee -a /etc/fstab

# Add disk to Longhorn
# Create the patch file
cat > longhorn-disk-patch.yaml <<'EOF'
spec:
  disks:
    sdb:
      allowScheduling: true
      evictionRequested: false
      path: /var/lib/longhorn/sdb
      storageReserved: 0
      tags: []
EOF

# Patch the LONGHORN node (not k8s node)
kubectl patch node.longhorn.io k8s-n1 -n longhorn-system --type merge --patch-file longhorn-disk-patch.yaml
kubectl patch node.longhorn.io k8s-n2 -n longhorn-system --type merge --patch-file longhorn-disk-patch.yaml
kubectl patch node.longhorn.io k8s-n3 -n longhorn-system --type merge --patch-file longhorn-disk-patch.yaml

# Verify disks
kubectl get nodes.longhorn.io -n longhorn-system -o yaml | grep -A 5 "path: /var/lib/longhorn/sdb"



## Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml

# Wait for pods
kubectl get pods -n metallb-system -w

# Configure IP address pool and L2 advertisement
cat > metallb-config.yaml <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: openstack-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.40-192.168.1.59
  avoidBuggyIPs: true
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: openstack-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - openstack-pool
  interfaces:
  - ens33  # Management interface
EOF

kubectl apply -f metallb-config.yaml

## Ceph Prep

# Create Ceph users with appropriate capabilities
ceph auth get-or-create client.glance \
  mon 'allow r' \
  osd 'allow class-read object_prefix rbd_children, allow rwx pool=glance.images' \
  -o /etc/ceph/ceph.client.glance.keyring

ceph auth get-or-create client.cinder \
  mon 'allow r' \
  osd 'allow class-read object_prefix rbd_children, allow rwx pool=cinder.volumes, allow rwx pool=vms, allow rx pool=glance.images' \
  -o /etc/ceph/ceph.client.cinder.keyring

ceph auth get-or-create client.cinder-backup \
  mon 'allow r' \
  osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups' \
  -o /etc/ceph/ceph.client.cinder-backup.keyring

ceph auth get-or-create client.nova \
  mon 'allow r' \
  osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rx pool=glance.images' \
  -o /etc/ceph/ceph.client.nova.keyring

# Extract keys for Kubernetes secrets
ceph auth get-key client.glance > client.glance.key
ceph auth get-key client.cinder > client.cinder.key
ceph auth get-key client.cinder-backup > client.cinder-backup.key
ceph auth get-key client.nova > client.nova.key

# Get FSID (you already have this: ad9e8f76-3aae-11f0-9498-0fcbb987ca56)
ceph fsid


ceph osd pool create cinder.volumes 128 128

# Create Cinder backups pool
ceph osd pool create cinder.backups 64 64

# Enable RBD on the pools
ceph osd pool application enable cinder.volumes rbd
ceph osd pool application enable cinder.backups rbd

# Grant access to the cinder pools
ceph auth caps client.cinder mon 'profile rbd' osd 'profile rbd pool=cinder.volumes, profile rbd pool=cinder.backups'

# Also check/fix the cinder-backup user
ceph auth get client.cinder-backup || ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=cinder.backups'

ceph auth get client.cinder






# Copy these keys and keyrings to the ctl machine

## Create OpenStack namespace
kubectl create namespace openstack

# Create ceph.conf ConfigMap
cat > ceph-config.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: ceph-etc
  namespace: openstack
data:
  ceph.conf: |
    [global]
    fsid = ad9e8f76-3aae-11f0-9498-0fcbb987ca56
    mon_host = [v2:10.10.28.10:3300/0,v1:10.10.28.10:6789/0],[v2:10.10.28.6:3300/0,v1:10.10.28.6:6789/0],[v2:10.10.28.7:3300/0,v1:10.10.28.7:6789/0],[v2:10.10.28.8:3300/0,v1:10.10.28.8:6789/0]
    auth_cluster_required = cephx
    auth_service_required = cephx
    auth_client_required = cephx
    
    [client]
    rbd_cache = true
    rbd_cache_writethrough_until_flush = true
    rbd_concurrent_management_ops = 20
EOF

kubectl apply -f ceph-config.yaml

# Create Ceph client secrets
kubectl create secret generic ceph-client-glance-key \
  --from-file=key=client.glance.key \
  --namespace=openstack

kubectl create secret generic ceph-client-cinder-key \
  --from-file=key=client.cinder.key \
  --namespace=openstack

kubectl create secret generic ceph-client-cinder-backup-key \
  --from-file=key=client.cinder-backup.key \
  --namespace=openstack

kubectl create secret generic ceph-client-nova-key \
  --from-file=key=client.nova.key \
  --namespace=openstack

# Create a properly formatted Ceph keyring secret
CEPH_KEY=$(kubectl get secret -n openstack ceph-client-cinder-key -o jsonpath='{.data.key}' | base64 -d)

kubectl create secret generic ceph-client-cinder-keyring -n openstack \
  --from-literal=cinder="[client.cinder]
    key = $CEPH_KEY" \
  --dry-run=client -o yaml | kubectl apply -f -

# Get the Ceph key value
CEPH_KEY=$(kubectl get secret -n openstack ceph-client-cinder-key -o jsonpath='{.data.key}' | base64 -d)

# Create a proper keyring format and patch the secret
kubectl patch secret -n openstack ceph-client-cinder-key -p "{\"data\":{\"cinder\":\"$(echo "[client.cinder]
    key = $CEPH_KEY" | base64 -w0)\"}}"


## Fix Glance Keys
# Get the current key value
CEPH_KEY=$(kubectl get secret ceph-client-glance-key -n openstack -o jsonpath='{.data.key}')

# Delete the old secret
kubectl delete secret ceph-client-glance-key -n openstack

# Recreate it with the correct key name "glance"
kubectl create secret generic ceph-client-glance-key -n openstack \
  --from-literal=glance=$(echo $CEPH_KEY | base64 -d)

# Verify it now has the "glance" key
kubectl get secret ceph-client-glance-key -n openstack -o jsonpath='{.data}' | jq 'keys'



## Generate UUID for libvirt secret
CINDER_SECRET_UUID=$(uuidgen)
echo $CINDER_SECRET_UUID > /tmp/cinder-secret-uuid

# Create libvirt secret definition
cat > /tmp/secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>$CINDER_SECRET_UUID</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF

# Define secret in libvirt
sudo apt install libvirt-clients -y
virsh secret-define --file /tmp/secret.xml

# Set secret value (use your ceph client.cinder key)
virsh secret-set-value --secret $CINDER_SECRET_UUID --base64 $(cat client.cinder.key)

#
user@k8s-ctl:~$ echo $CINDER_SECRET_UUID
7e733988-7a86-4790-9655-1432a83e64dc


## OVN networking bridge configuration
# Create provider bridge for ens35 (trunk port)
sudo apt install -y openvswitch-switch
sudo ovs-vsctl add-br br-provider
sudo ovs-vsctl set bridge br-provider protocols=OpenFlow13
sudo ovs-vsctl add-port br-provider ens35

# Configure bridge mapping
sudo ovs-vsctl set Open_vSwitch . \
  external-ids:ovn-bridge-mappings="provider:br-provider"

# Set chassis MAC mappings (prevents MAC jumping)
# Use unique MAC per node
sudo ovs-vsctl set Open_vSwitch . \
  external-ids:ovn-chassis-mac-mappings="provider:fa:16:3e:00:01:01"  # k8s-n1
# Change last octet for k8s-n2 (:02) and k8s-n3 (:03)

# Configure OVN encapsulation
sudo ovs-vsctl set Open_vSwitch . \
  external-ids:ovn-encap-type=geneve
sudo ovs-vsctl set Open_vSwitch . \
  external-ids:ovn-encap-ip=192.168.1.26  # Change per node

# Enable gateway chassis capability
sudo ovs-vsctl set Open_vSwitch . \
  external-ids:ovn-cms-options="enable-chassis-as-gw"

########## Snap VMs
$snapname = 'Yaook Try 1 - Pre Yaook Install'
$description = get-date
get-vm 'k8s-ctl','k8s-n1','k8s-n2','k8s-n3' | new-snapshot -Name $snapname -Description $description -Quiesce -Memory
##########

# Prevent stuff from running on my control node 
kubectl cordon k8s-ctl
kubectl drain k8s-ctl --ignore-daemonsets --delete-emptydir-data

########################## Yaook Installation

# Create working directory
mkdir -p ~/openstack-deployment
cd ~/openstack-deployment

# Clone Yaook operator repository
git clone https://gitlab.com/yaook/operator.git yaook-operator
cd yaook-operator
sudo apt install -y python3.11-dev libffi-dev libssl-dev libxml2-dev libxslt1-dev libpq-dev libjpeg-dev build-essential pkg-config

# Set up Python environment
cd ~/openstack-deployment/yaook-operator
# Check out version 
git checkout 0.20250724.0
python3.11 -m venv venv
source venv/bin/activate
python --version  # Must show Python 3.11.x
pip install --upgrade pip

# Install CUE language tool (for configuration validation)
# Download from https://github.com/cue-lang/cue/releases
cd /tmp
wget https://github.com/cue-lang/cue/releases/download/v0.4.3/cue_v0.4.3_linux_amd64.tar.gz
tar -xzf cue_v0.4.3_linux_amd64.tar.gz
sudo mv cue /usr/local/bin/
sudo chmod +x /usr/local/bin/cue

# Verify installation
cue version
cd ~/openstack-deployment/yaook-operator

# Install Requirements
pip install -r requirements-build.txt
pip install -e .
export MYPYPATH="$(pwd)/.local/mypy-stubs"
export YAOOK_OP_NAMESPACE="yaook"
export YAOOK_OP_CLUSTER_DOMAIN="cluster.local"

# Step D: Generate type stubs
stubgen -p kubernetes_asyncio -o "$MYPYPATH"

# Step E: NOW run make all
make all

# Deploy CRDs
export NAMESPACE=openstack
helm upgrade --install --namespace $NAMESPACE yaook-crds ./yaook/helm_builder/Charts/crds


## Install cert-manager 
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml

# Generate CA certificate for OpenStack services
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt \
  -subj "/CN=OpenStack-CA/O=region.com"

# Create CA secret
kubectl create secret tls ca-key-pair \
  --cert=ca.crt --key=ca.key \
  --namespace=$NAMESPACE

# Create CA issuer
cat > ca-issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: openstack
spec:
  ca:
    secretName: ca-key-pair
EOF

kubectl apply -f ca-issuer.yaml

# Create self-signed issuer for internal services
cat > selfsigned-issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: openstack
spec:
  selfSigned: {}
EOF

kubectl apply -f selfsigned-issuer.yaml


## Deploy infrastructure operator (manages MySQL, RabbitMQ, Memcached)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-operator-crds prometheus-community/prometheus-operator-crds

# Verify the CRDs are installed
kubectl get crd | grep monitoring.coreos.com


# Label all worker nodes for API and infrastructure services
kubectl label nodes k8s-n1 k8s-n2 k8s-n3 \
  any.yaook.cloud/api=true \
  infra.yaook.cloud/any=true \
  infra.yaook.cloud/db=true \
  infra.yaook.cloud/mq=true \
  infra.yaook.cloud/caching=true

# Label for compute services
kubectl label nodes k8s-n1 k8s-n2 k8s-n3 \
  compute.yaook.cloud/hypervisor=kvm

# Label for network services
kubectl label nodes k8s-n1 k8s-n2 k8s-n3 \
  network.yaook.cloud/neutron-ovn-agent=true

# This section will be required when we have control-plane only nodes. This is hyperconverged at this time.
## Add taints to ensure compute workloads only run on labeled nodes
#kubectl taint nodes k8s-n1 k8s-n2 k8s-n3 \
#  compute.yaook.cloud/hypervisor=:NoSchedule

## BUG, hardcoded ceph addresses in the cinder-operator.   Prevents cinder storage from mounting.
vi yaook/op/cue/pkg/yaook.cloud/ceph_config_by_yaook/defaults.cue
#
package ceph_config_by_yaook

ceph_conf_spec: {
}
# EOF
##

## BUG, keyfile vs keyring mismatch. Prevents cinder storage from mounting.
# Backup first
cp yaook/op/cinder/__init__.py yaook/op/cinder/__init__.py.bak

# Fix the bug - change keyfile to keyring
sed -i 's/"keyfile":/"keyring":/' yaook/op/cinder/__init__.py

# Verify the change
sed -n '105,120p' yaook/op/cinder/__init__.py
##


## Build the operator image
sudo apt  install docker.io -y
which crictl
which ctr
sudo usermod -aG docker $USER
newgrp docker
docker version
docker build -t yaook/operator:2024.1-stable \
  --build-arg cue_version=0.4.3 \
  --build-arg ovs_version=branch-3.3 \
  --build-arg TARGETPLATFORM=linux/amd64 \
  .
# Wait 10m

## Start a local registry on k8s-ctl
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# Tag and push the image
docker tag yaook/operator:2024.1-stable 192.168.1.24:5000/yaook/operator:2024.1-stable
docker push 192.168.1.24:5000/yaook/operator:2024.1-stable


# On each worker node (k8s-n1, k8s-n2, k8s-n3), configure insecure registry
# Add to /etc/docker/daemon.json (or create it):

sudo bash -c 'cat > /etc/docker/daemon.json <<EOF
{
  "insecure-registries": ["192.168.1.24:5000"]
}
EOF'
sudo apt  install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
docker version
sudo systemctl enable docker
sudo systemctl restart docker
sudo docker pull 192.168.1.24:5000/yaook/operator:2024.1-stable


# Configure containerd on k8s-n1 (and then all nodes)
ssh k8s-n1 'sudo bash -c "cat >> /etc/containerd/config.toml << EOF

[plugins.\"io.containerd.grpc.v1.cri\".registry.mirrors.\"192.168.1.24:5000\"]
  endpoint = [\"http://192.168.1.24:5000\"]

[plugins.\"io.containerd.grpc.v1.cri\".registry.configs.\"192.168.1.24:5000\".tls]
  insecure_skip_verify = true
EOF
"'

# Restart containerd
ssh k8s-n1 'sudo systemctl restart containerd'

# Test with crictl
ssh k8s-n1 'sudo crictl pull 192.168.1.24:5000/yaook/operator:2024.1-stable'

# If that works, apply to all nodes
for node in k8s-n2 k8s-n3; do
  echo "Configuring $node..."
  ssh $node 'sudo bash -c "cat >> /etc/containerd/config.toml << EOF

[plugins.\"io.containerd.grpc.v1.cri\".registry.mirrors.\"192.168.1.24:5000\"]
  endpoint = [\"http://192.168.1.24:5000\"]

[plugins.\"io.containerd.grpc.v1.cri\".registry.configs.\"192.168.1.24:5000\".tls]
  insecure_skip_verify = true
EOF
"'
  ssh $node 'sudo systemctl restart containerd'
done

# Add the required labels to your worker nodes
kubectl label nodes k8s-n1 operator.yaook.cloud/infra=true
kubectl label nodes k8s-n2 operator.yaook.cloud/infra=true
kubectl label nodes k8s-n3 operator.yaook.cloud/infra=true

# Also add the "any" label
kubectl label nodes k8s-n1 operator.yaook.cloud/any=true
kubectl label nodes k8s-n2 operator.yaook.cloud/any=true
kubectl label nodes k8s-n3 operator.yaook.cloud/any=true

# Label nodes for Cinder scheduler (and other Cinder services)
kubectl label nodes k8s-n1 k8s-n2 k8s-n3 block-storage.yaook.cloud/cinder-any-service=true


## Deploy the operators

helm upgrade --install --namespace openstack \
  infra-operator ./yaook/helm_builder/Charts/infra-operator/ \
  --set image.repository=192.168.1.24:5000/yaook/operator \
  --set image.tag=2024.1-stable \
  --set image.pullPolicy=Always \
  --wait --timeout=600s

kubectl set image deployment/infra-operator -n openstack \
  operator=192.168.1.24:5000/yaook/operator:2024.1-stable

# If that works, do them all

# Deploy OpenStack service operators
for OP_NAME in keystone glance nova neutron-ovn cinder horizon neutron-operator keystone-resources-operator  keystone-controller; do
  helm upgrade --install --namespace $NAMESPACE \
    ${OP_NAME}-operator ./yaook/helm_builder/Charts/${OP_NAME}-operator/ \
    --set operator.image.repository=192.168.1.24:5000/yaook/operator \
    --set operator.image.tag=local \
    --set operator.image.pullPolicy=Always
done

# Update all operators at once
for operator in infra-operator keystone-operator glance-operator nova-operator cinder-operator neutron-ovn-operator horizon-operator neutron-operator keystone-resources-operator  keystone-controller; do
  echo "Updating $operator..."
  kubectl set image deployment/$operator -n openstack \
    operator=192.168.1.24:5000/yaook/operator:2024.1-stable
Done


# Keystone Resources Operator - IDK Why this one didn’t go with the above 'foreach' loop
helm package ./yaook/helm_builder/Charts/keystone-resources-operator
helm upgrade --install keystone-resources-operator \
  ./keystone-resources-operator-*.tgz \
  --namespace openstack \
  --set image.repository=192.168.1.24:5000/yaook/operator \
  --set image.tag=2024.1-stable

 kubectl set image deployment/keystone-resources-operator -n openstack \
    operator=192.168.1.24:5000/yaook/operator:2024.1-stable


# Neutron Operator
helm package ./yaook/helm_builder/Charts/neutron-operator
helm upgrade --install neutron-operator \
  ./neutron-operator-*.tgz \
  --namespace openstack \
  --set image.repository=192.168.1.24:5000/yaook/operator \
  --set image.tag=2024.1-stable

 kubectl set image deployment/neutron-operator -n openstack \
    operator=192.168.1.24:5000/yaook/operator:2024.1-stable


# Verify all images updated
echo -e "\nVerifying images:"
kubectl get deployments -n openstack -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image | grep operator


########## Snap VMs
$snapname = 'Yaook Try 2 - Pre Keystone'
$description = get-date
get-vm 'k8s-ctl','k8s-n1','k8s-n2','k8s-n3' | new-snapshot -Name $snapname -Description $description -Quiesce -Memory
##########

# Tag memcached image on all nodes
for node in k8s-n1 k8s-n2 k8s-n3; do
  echo "Tagging on $node..."
  ssh $node 'sudo ctr -n k8s.io images tag docker.io/bitnami/memcached:latest docker.io/bitnami/memcached:1.6.38-debian-12-r8'
done

# Use official mariadb and retag on all nodes
for node in k8s-n1 k8s-n2 k8s-n3; do
  echo "Processing $node..."
  ssh $node 'sudo ctr -n k8s.io images pull public.ecr.aws/bitnami/mariadb-galera:11.4 && sudo ctr -n k8s.io images tag public.ecr.aws/bitnami/mariadb-galera:11.4 docker.io/bitnami/mariadb-galera:11.4.7-debian-12-r3'
done


## Deploy Keystone
kubectl apply -f ~/openstack-deployment/keystone.yaml


## Deploy Glance
kubectl apply -f ~/openstack-deployment/glance.yaml


### Deploy Nova + memcache + placement 
# Generate UUID for libvirt secret
NOVA_LIBVIRT_UUID=$(uuidgen)
echo "Replace Nova libvirt UUID in nova.yaml: $NOVA_LIBVIRT_UUID"

# Pull memcached from Docker Hub (if latest exists) or try a different tag
docker pull memcached:1.6
docker tag memcached:1.6 bitnami/memcached:1.6.38-debian-12-r8

# Distribute to all nodes
for node in k8s-n1 k8s-n2 k8s-n3; do
  echo "Processing $node..."
  ssh $node 'sudo ctr -n k8s.io images pull docker.io/library/memcached:1.6 && \
             sudo ctr -n k8s.io images tag docker.io/library/memcached:1.6 docker.io/bitnami/memcached:1.6.38-debian-12-r8'
done

# Apply
kubectl apply -f ~/openstack-deployment/nova.yaml

# Label the compute nodes
kubectl label nodes k8s-n1 compute.yaook.cloud/enabled=true
kubectl label nodes k8s-n2 compute.yaook.cloud/enabled=true
kubectl label nodes k8s-n3 compute.yaook.cloud/enabled=true



### Deploy Neutron
kubectl apply -f ~/openstack-deployment/neutron.yaml


## Configure Ingress
# Install NGINX Ingress Controller using kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/baremetal/deploy.yaml

# Wait for it to start
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Check the service
kubectl get svc -n ingress-nginx

# Patch the service to use NodePort or LoadBalancer
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec":{"type":"LoadBalancer"}}'

# Get the IP 
kubectl get svc ingress-nginx-controller -n ingress-nginx


##Neutron Fixes Remove openvswitch from hosts
# On each worker node (k8s-n1, k8s-n2, k8s-n3):
ssh k8s-n1 "sudo systemctl stop openvswitch-switch ovs-vswitchd ovsdb-server"
ssh k8s-n1 "sudo systemctl disable openvswitch-switch ovs-vswitchd ovsdb-server"

ssh k8s-n2 "sudo systemctl stop openvswitch-switch ovs-vswitchd ovsdb-server"
ssh k8s-n2 "sudo systemctl disable openvswitch-switch ovs-vswitchd ovsdb-server"

ssh k8s-n3 "sudo systemctl stop openvswitch-switch ovs-vswitchd ovsdb-server"
ssh k8s-n3 "sudo systemctl disable openvswitch-switch ovs-vswitchd ovsdb-server"


### Healthcheck openstack 
# List Service Endpoints
openstack endpoint list

# Service List
openstack service list

# Compute Healthcheck
openstack compute service list

# Network Healthcheck
openstack network agent list

# Storage Healthcheck
openstack volume service list

# Image Healthcheck
openstack image list

# Show back end storage services
openstack volume service list


########## Snap VMs
$snapname = 'Yaook Try 2 - Pre Cinder'
$description = get-date
get-vm 'k8s-ctl','k8s-n1','k8s-n2','k8s-n3' | new-snapshot -Name $snapname -Description $description -Quiesce -Memory
##########

# Install Clients 
sudo apt  install python3-openstackclient -y



### Cinder

# Label all worker nodes for Cinder services
kubectl label node k8s-n1 block-storage.yaook.cloud/cinder-any-service=true
kubectl label node k8s-n2 block-storage.yaook.cloud/cinder-any-service=true
kubectl label node k8s-n3 block-storage.yaook.cloud/cinder-any-service=true

# Apply it
kubectl apply -f ~/openstack-deployment/cinder.yaml

# Verify
openstack --insecure volume service list


########## Snap VMs
$snapname = 'Yaook Try 2 - Post Cinder'
$description = get-date
get-vm 'k8s-ctl','k8s-n1','k8s-n2','k8s-n3' | new-snapshot -Name $snapname -Description $description -Quiesce -Memory
##########


## Nova compute operator
kubectl apply -f /tmp/nova-compute-operator.yaml

kubectl apply -f ~/openstack-deployment/nova-fixed.yaml

KEY_VALUE=$(kubectl get secret ceph-client-nova-key -n openstack -o jsonpath='{.data.key}')

# Patch the secret to add the "nova" key
kubectl patch secret ceph-client-nova-key -n openstack --type=json -p="[
  {
    \"op\": \"add\",
    \"path\": \"/data/nova\",
    \"value\": \"$KEY_VALUE\"
  }
]"

# Verify both keys exist
kubectl get secret ceph-client-nova-key -n openstack -o jsonpath='{.data}' | jq

# The pods should start mounting now - check
sleep 30
kubectl get pods -n openstack -l state.yaook.cloud/component=compute


# Wait and check
sleep 30
kubectl get pods -n openstack | grep nova-compute-operator
kubectl logs -n openstack deployment/nova-compute-operator --tail=50


### Horizon
kubectl apply -f ~/openstack-deployment/horizon.yaml

# Create DNS record for horizon.region.com

########## Snap VMs
$snapname = 'Yaook Try 3 - Post Horizon'
$description = get-date
get-vm 'k8s-ctl','k8s-n1','k8s-n2','k8s-n3' | new-snapshot -Name $snapname -Description $description -Quiesce -Memory
##########

# Get all admin credentials
kubectl get secret keystone-admin -n openstack -o yaml

# Or just the important fields
echo "Username: $(kubectl get secret keystone-admin -n openstack -o jsonpath='{.data.OS_USERNAME}' | base64 -d)"
echo "Password: $OS_PASSWORD"
echo "Project: $(kubectl get secret keystone-admin -n openstack -o jsonpath='{.data.OS_PROJECT_NAME}' | base64 -d)"
echo "Domain: $(kubectl get secret keystone-admin -n openstack -o jsonpath='{.data.OS_USER_DOMAIN_NAME}' | base64 -d)"


## Test it 
https://horizon.region.com/
Username: yaook-sys-maint
Password: BTN1olY4Fcd0VzxsLjY83ffeoIIZ4WwIRpPCCo-pKkM
Project: admin
Domain: Default



## Glance Troubleshooting
kubectl get secret ceph-client-glance-key -n openstack -o yaml

# Get your Ceph key from your Ceph cluster
ceph auth get-or-create client.glance

# Then update the secret with the actual key value
kubectl create secret generic ceph-client-glance-key \
  --from-literal=glance='<your-ceph-key-here>' \
  -n openstack --dry-run=client -o yaml | kubectl apply -f -

# After updating the secret, restart Glance again:
kubectl rollout restart deployment glance-api -n openstack



## Unsigned cert workaround 
## Bypass insecure flag
cat >> ~/admin-openrc <<'EOF'

# Always use --insecure flag
alias openstack='command openstack --insecure'
EOF

source ~/admin-openrc
openstack image list


### Octavia 


# Deploy Operator
helm install octavia-operator \
  ~/openstack-deployment/yaook-operator/yaook/helm_builder/Charts/octavia-operator/octavia-operator-0.20250724.0-base.tgz \
  -n openstack

kubectl set image deployment/octavia-operator -n openstack \
  operator=192.168.1.24:5000/yaook/operator:2024.1-stable


# Deploy it
kubectl apply -f ~/openstack-deployment/octavia.yaml


kubectl set image statefulset/octavia-octavia-db -n openstack \
  galera=192.168.1.24:5000/bitnami/mariadb-galera:11.4.7-debian-12-r3

# Delete the pod to recreate with the correct image
kubectl delete pod octavia-octavia-db-0 -n openstack

kubectl set image statefulset/octavia-octavia-db -n openstack \
  mariadb-galera=192.168.1.24:5000/bitnami/mariadb-galera:11.4.7-debian-12-r3

# Delete the pod to recreate
kubectl delete pod octavia-octavia-db-0 -n openstack



# Bitnami workaround - Tag and push the image
docker pull bitnamilegacy/mariadb-galera:11.0.6-debian-12-r0
docker tag bitnamilegacy/mariadb-galera:11.0.6-debian-12-r0 192.168.1.24:5000/yaook/mariadb-galera:11.0.6-debian-12-r0
docker push 192.168.1.24:5000/yaook/mariadb-galera:11.0.6-debian-12-r0
