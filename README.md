Description:
Yaook 2024.1 (Caracal)
K8s v1.31.13 (Kubespray)
Longhorn 
Calico CNI
Core Services + OVN Networking
External Ceph Integration (Squid 19.2.3)
MetalLB Loadbalancer 
Octavia
Horizon

Not supported at this time (10-25):
Masakari
Watcher
Skyline 


Machine Inventory:
Name    vCPU RAM(GB) HDD(GB) IPv4
k8s-ctl    2    4.00   60.00 192.168.1.24
k8s-n1     8   16.00  310.00 192.168.1.26, 10.233.97.192
k8s-n2     8   16.00  310.00 192.168.1.27, 10.233.122.64
k8s-n3     8   16.00  310.00 192.168.1.28, 10.233.121.192

NIC Layout:
ens33      LAN  192.168.1.0/24 static IP Assigned
ens35      Trunk 27/28 No IP assigned.

Findings:
    • “Yaook” stands for “Yet Another OpenStack on Kubernetes.”
    • No image repository exists for operator?  Had to build images locally.
    • Yaook Operators for each major service are deployed first from a compiled image, and are a prerequisite for the service deployment. 
    • Yaook currently does not support Masakari, Watcher, or Skyline. 
    • Yaook does not use Helm charts for OpenStack services—instead, it deploys custom Kubernetes operators that watch Custom Resources (CRs). Each OpenStack service (Keystone, Nova, Neutron, etc.) has a dedicated operator managing its lifecycle through a state machine pattern. Operators installed via Helm, but OpenStack services deployed via CRs.


