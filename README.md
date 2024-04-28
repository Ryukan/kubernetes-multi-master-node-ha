# Set up on Debian a Highly Available Kubernetes Cluster using kubeadm
Follow this documentation to set up a highly available Kubernetes cluster using __Debian 12__.

This documentation guides you in setting up a cluster with two master nodes, two worker nodes and a load balancer node using HAProxy.

## Environment
|Role|FQDN|IP|RAM|CPU|API Version|
|----|----|----|----|----|----|
|HAproxy|lb.kubernetes.local|10.5.200.40|1G|1|
|Master|master1.kubernetes.local|10.5.200.41|2G|2|v1.30.0|
|Master|master2.kubernetes.local|10.5.200.42|2G|2|v1.30.0|
|Worker|worker1.kubernetes.local|10.5.200.43|4G|1|v1.30.0|
|Worker|worker2.kubernetes.local|10.5.200.44|4G|1|v1.30.0|



> * Perform all the commands as root user unless otherwise specified

## Set hostname on Each Node
Login to to master node and set hostname via hostnamectl command,
```
sudo hostnamectl set-hostname "master1"
sudo hostnamectl set-hostname "master2"
exec bash
```
On the worker nodes, run
```
sudo hostnamectl set-hostname "worker1"
sudo hostnamectl set-hostname "worker2"
exec bash
```
On the loadbalancer, run
```
sudo hostnamectl set-hostname "lb"
exec bash
```

## Set up hostnames
Add the following lines in /etc/hosts file on each node
```
10.5.200.40     lb.kubernetes.local          lb
10.5.200.41     master1.kubernetes.local     master1
10.5.200.42     master2.kubernetes.local     master2
10.5.200.43     worker1.kubernetes.local     worker1
10.5.200.44     worker2.kubernetes.local     worker2
```

## Configure Static IP
On each node set static IP:
```
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
```
```
sudo nano /etc/network/interfaces
```
```
auto ens32
iface ens32 inet static
address 10.5.200.xx
netmask 255.255.255.0
gateway 10.5.200.254
dns-nameservers 8.8.4.4 8.8.8.8
```
```
systemctl restart networking
```

## Set up load balancer node
##### Install Haproxy
```
apt update && apt install -y haproxy
```
##### Configure haproxy
Append the below lines to **/etc/haproxy/haproxy.cfg**
```
frontend kubernetes-frontend
    bind 10.5.200.40:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server master1 10.0.5.41:6443 check fall 3 rise 2
    server master2 10.0.5.42:6443 check fall 3 rise 2
```
##### Restart haproxy service
```
systemctl restart haproxy
```

## On all kubernetes nodes (master1, master2, worker1, worker2)
##### Disable Firewall
```
ufw disable
```
##### Disable swap
```
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
##### Update sysctl settings for Kubernetes networking
```
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```
```
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
```
sudo sysctl --system
```
##### Install Containerd engine
Install Prerequisites
```
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg
```
Add Docker’s GPG Repo Key
```
sudo curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg
```
Add the Docker Repo to Debian 12
```
sudo echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/debian bookworm stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Install Containerd
```
sudo apt update

sudo apt install -y containerd.io
```
Configure containerd to start using systemd as cgroup:
```
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
Restart and enable the containerd service:
```
sudo systemctl restart containerd
sudo systemctl enable containerd
```
### Kubernetes Setup
##### Add Apt Repository for Kubernetes (all nodes)
Update the apt package index and install packages needed to use the Kubernetes apt repository:
```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
Download the public signing key for the Kubernetes package repositories
```
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
Add the appropriate Kubernetes apt repository
```
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
sudo echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Enable the kubelet service
```
sudo systemctl enable --now kubelet
```

## On any one of the Kubernetes master node (Eg: master1)
##### Initialize Kubernetes Cluster
```
kubeadm init --control-plane-endpoint="10.5.200.40:6443" --upload-certs --apiserver-advertise-address=10.5.200.40
```
Copy the commands to join other master nodes and worker nodes.
##### Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml
```

## Join other nodes to the cluster (master2 & worker1, worker2)
> Use the respective kubeadm join commands you copied from the output of kubeadm init command on the first master.

> IMPORTANT: You also need to pass --apiserver-advertise-address to the join command when you join the other master node.

## Downloading kube config to your local machine
On your host machine
```
mkdir ~/.kube
scp root@10.5.200.41:/etc/kubernetes/admin.conf ~/.kube/config
```

## Verifying the cluster
```
kubectl cluster-info
kubectl get nodes
kubectl get cs
```


