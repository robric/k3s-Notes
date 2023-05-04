# kube install

Vanilla install with flannel v1.25.

```
#!/bin/bash

apt update
apt install net-tools -y
apt install docker.io -y
systemctl start docker
systemctl enable docker
apt install apt-transport-https curl -y
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main" -y
apt install kubelet=1.25.4-00 kubeadm=1.25.4-00 kubectl=1.25.4-00 kubernetes-cni -y
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
kubeadm init --pod-network-cidr 10.244.0.0/16
sleep 30
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```
Multus
```
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
```
SCTP tooling
```
sudo apt install lksctp-tools
```
SCTP is by default in Ubuntu 20.04.5 LTS (no sctp kernel module)
```console
ubuntu@ip-172-31-23-42:~$ checksctp
SCTP supported
ubuntu@ip-172-31-23-42:~$
```
# k3s-Notes

Quick notes k3S installation


## CNI specifics

CNI configurations (usually /etc/cni) are located in /var/lib/rancher/k3s/agent/etc/cni/net.d
CNI binaries (usually /opt/cni/bin) are located in var/lib/rancher/k3s/data/current/bin

## Multus

Installation here for tweaking multus install with appropriate folders: https://gist.github.com/janeczku/ab5139791f28bfba1e0e03cfc2963ecf

## auxialiary plugins

Here how to compile and install diverse plugins (mac-vlan):  https://medium.com/@sarpkoksal/multus-installation-on-kubernetes-a0a6ef04972f

```
git clone https://github.com/containernetworking/plugins.git
cd plugins
ls
./build_linux.sh
ls
cp bin/macvlan /var/lib/rancher/k3s/data/current/bin/
cp bin/static /var/lib/rancher/k3s/data/current/bin/
cp bin/ipvlan /var/lib/rancher/k3s/data/current/bin/
```
Edit macvlan.conf
```
root@ip-10-0-1-60:/var/lib/rancher/k3s/agent/etc/cni/net.d# cat 20-macvlan.conf 
{
    "cniVersion": "0.4.0",
    "name": "macvlan-network",
    "type": "macvlan",
    "mode": "bridge",
}
```

# Minikube install with driver none 

On Ubuntu

```
#!/bin/bash

sudo apt install net-tools iptables iproute2 conntrack lksctp-tools -y
#sudo snap install go --classic
## Install docker 
sudo apt-get update -y
sudo apt-get -y install \
    ca-certificates \
    curl \
    gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get -y update
sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
## Setup environment for minikube
git clone https://github.com/Mirantis/cri-dockerd.git

# Run these commands as root
###Install GO###
wget https://storage.googleapis.com/golang/getgo/installer_linux
chmod +x ./installer_linux
./installer_linux
source ~/.bash_profile

cd cri-dockerd
mkdir bin
go build -o bin/cri-dockerd
mkdir -p /usr/local/bin
sudo install -o root -g root -m 0755 bin/cri-dockerd /usr/local/bin/cri-dockerd
sudo cp -a packaging/systemd/* /etc/systemd/system
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
## CRICTL
cd ~/
VERSION="v1.26.0" # check latest version in /releases page
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
## Install kubectl 
cd ~/
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
## Install minikube
cd ~/
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
## Create cluster
sudo apt-get install -y conntrack
cd ~/
sudo -E minikube start  --driver=none --cni=calico
# Configure kubectl
cd ~/
sudo mv /home/ubuntu/.kube /home/ubuntu/.minikube $HOME
sudo chown -R $USER $HOME/.kube $HOME/.minikube
# Configure CNI
cd ~/
git clone https://github.com/containernetworking/plugins
cd plugins && ./build_linux.sh
sleep 15
sudo cp -r bin/* /opt/cni/bin/
## Install multus
cd ~/
git clone https://github.com/k8snetworkplumbingwg/multus-cni.git
cat multus-cni/deployments/multus-daemonset-thick.yml | kubectl apply -f -
kubectl get pods --all-namespaces | grep multus
for i in `seq 1 10`
do
	health=$(kubectl get pods --all-namespaces | grep multus | grep -o Running)
	if [ $health == "Running" ];
	 then
		break
	 else
		echo "Waiting for multus to be installed"
	fi
	sleep 5
done
## Configure kubectl with completion
sudo apt-get install bash-completion -y
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
#Install helm and helm spray
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
helm completion bash | sudo tee /etc/bash_completion.d/helm > /dev/null
```

# AWS fixes 
```
aws ec2 modify-network-interface-attribute  --network-interface-id eni-0307b17e9fc696925 --no-source-dest-check
```
