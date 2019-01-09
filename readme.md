
# flannel

Installation: https://github.com/coreos/flannel/blob/master/Documentation/running.md

```
wget https://github.com/coreos/flannel/releases/download/v0.10.0/flanneld-amd64 && chmod +x flanneld-amd64

sudo ./flanneld-amd64 # it will hang waiting to talk to etcd

docker run --rm --net=host quay.io/coreos/etcd

docker run --rm --net=host quay.io/coreos/etcd etcdctl set /coreos.com/network/config '{ "Network": "10.5.0.0/16", "Backend": {"Type": "vxlan"}}'
```

Edit and restart docker:

```
# /sudo:root@zz:/usr/lib/systemd/system/docker.service

#Replace
#ExecStart=/usr/bin/dockerd -H fd://

EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
```
