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
