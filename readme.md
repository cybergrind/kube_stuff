
# flannel (local installation for docker)

Installation: https://github.com/coreos/flannel/blob/master/Documentation/running.md

Other: https://docker-k8s-lab.readthedocs.io/en/latest/docker/docker-flannel.html

```
wget https://github.com/coreos/flannel/releases/download/v0.10.0/flanneld-amd64 && chmod +x flanneld-amd64

sudo ./flanneld-amd64 # it will hang waiting to talk to etcd
# OR
sudo ./flanneld-amd64 -etcd-endpoints=http://192.168.88.34:2379

docker run --rm --net=host quay.io/coreos/etcd
# OR
docker run --rm --net=host quay.io/coreos/etcd etcd --listen-client-urls 'http://0.0.0.0:2379' --advertise-client-urls=http://192.168.88.34:2379

docker run --rm --net=host quay.io/coreos/etcd etcdctl set /coreos.com/network/config '{ "Network": "10.5.0.0/16", "Backend": {"Type": "vxlan"}}'
```

Edit and restart docker:

```
# /sudo:root@localhost:/usr/lib/systemd/system/docker.service
# (find-file "/sudo:root@localhost:/usr/lib/systemd/system/docker.service")

#Replace
#ExecStart=/usr/bin/dockerd -H fd://

EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
```


# kubernetes

```
# (find-file "/sudo:root@localhost:/usr/lib/systemd/system/kubelet.service")
kubeadm init --pod-network-cidr=10.244.0.0/16
# OR
KUBELET_EXTRA_ARGS=--cgroup-driver=cgroupfs kubeadm init --pod-network-cidr=10.244.0.0/16

# reset cluster if something is wrong
# kubeadm reset
```

`kubectl get cs`

```
# (find-file "/sudo:root@localhost:/etc/docker/daemon.json")
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}

```
