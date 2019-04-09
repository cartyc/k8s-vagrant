# -*- mode: ruby -*-
# vi: set ft=ruby :

# This script to install Kubernetes will get executed after we have provisioned the box 
$script = <<-SCRIPT


# Install kubernetes
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y libseccomp2

# Download Containerd
export VERSION=1.1.0
wget https://storage.googleapis.com/cri-containerd-release/cri-containerd-${VERSION}.linux-amd64.tar.gz
sudo tar --no-overwrite-dir -C / -xzf cri-containerd-${VERSION}.linux-amd64.tar.gz

# Checksums
# sha256sum cri-containerd-${VERSION}.linux-amd64.tar.gz
# curl https://storage.googleapis.com/cri-containerd-release/cri-containerd-${VERSION}.linux-amd64.tar.gz.sha256

# Install Containerd
echo Installing Containerd
sudo tar --no-overwrite-dir -C / -xzf cri-containerd-${VERSION}.linux-amd64.tar.gz
sudo systemctl start containerd

# Install k8s components
apt-get install -y kubelet kubeadm kubectl libseccomp2

cat <<EOF >/etc/systemd/system/kubelet.service.d/0-containerd.conf
[Service]                                                 
Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
EOF

modprobe br_netfilter
echo '1' > /proc/sys/net/ipv4/ip_forward

systemctl daemon-reload

# kubelet requires swap off
sudo swapoff -a
# keep swap off after reboot
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# sed -i '/ExecStart=/a Environment="KUBELET_EXTRA_ARGS=--cgroup-driver=cgroupfs"' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# sed -i '0,/ExecStart=/s//Environment="KUBELET_EXTRA_ARGS=--cgroup-driver=cgroupfs"\\n&/' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# Get the IP address that VirtualBox has given this VM
IPADDR=`ifconfig eth1 | grep -i Mask | awk '{print $2}'| cut -f2 -d:`
echo This VM has IP address $IPADDR

# Set up Kubernetes
NODENAME=$(hostname -s)
kubeadm init --apiserver-cert-extra-sans=$IPADDR  --node-name $NODENAME --pod-network-cidr=10.244.0.0/16

# Set up admin creds for the vagrant user
echo Copying credentials to /home/vagrant...
sudo --user=vagrant mkdir -p /home/vagrant/.kube
cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

export KUBECONFIG=/home/vagrant/.kube/config

kubectl taint nodes --all node-role.kubernetes.io/master-

// Calico v3.3
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

echo Machine IP is $IPADDR\n
cat /home/vagrant/.kube/config

SCRIPT

Vagrant.configure("2") do |config|
  config.vm.provider "virtualbox" do |v|
    v.memory = 16384
    v.cpus = 2
  config.vm.network "forwarded_port", guest: 9090, host: 9090
  config.vm.network "forwarded_port", guest: 3000, host: 3000
  config.vm.network "forwarded_port", guest: 12345, host: 12345
  end
  
  # Specify your hostname if you like
  # config.vm.hostname = "name"
  config.vm.box = "bento/ubuntu-18.04"
  config.vm.network "private_network", type: "dhcp"
  # config.vm.provision "docker"
  # Specify the shared folder mounted from the host if you like
  # By default you get "." synced as "/vagrant"
  # config.vm.synced_folder ".", "/folder"  
  config.vm.provision "shell", inline: $script
end