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

