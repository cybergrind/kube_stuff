
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

(find-file "/sudo:root@localhost:/etc/kubernetes")

```
# (find-file "/sudo:root@localhost:/usr/lib/systemd/system/kubelet.service")
kubeadm init --pod-network-cidr=10.244.0.0/16
# OR
KUBELET_EXTRA_ARGS=--cgroup-driver=cgroupfs kubeadm init --pod-network-cidr=10.244.0.0/16

# reset cluster if something is wrong
# kubeadm reset
```


Note: you should use the same cgroupdriver. Either `systemd` or `cgroupfs`

```
# (find-file "/sudo:root@localhost:/etc/docker/daemon.json")
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}

# OR

ExecStart=/usr/bin/dockerd -H fd:// --exec-opt native.cgroupdriver=systemd
```


```
# /etc/kubernetes/kubelet
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
KUBELET_HOSTNAME="--hostname-override=zz"

KUBELET_KUBECONFIG="--kubeconfig=/etc/kubernetes/kubelet.kubeconfig"

KUBELET_ARGS="--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=systemd --pod-infra-container-image=k8s.gcr.io/pause:3.1"
```

#### Check status

```
kubectl get cs
kubectl get nodes

kubectl get pods --all-namespaces
```


#### Arch specific

Required packages:

* kubernetes
* cni
* cni-plugins (flannel)


```
systemctl disable kube-apiserver
systemctl disable kube-controller-manager
systemctl disable kube-scheduler
systemctl disable kube-proxy
```


```
# (find-file "/sudo:root@localhost:/usr/lib/systemd/system/kubelet.service")
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://kubernetes.io/docs/concepts/overview/components/#kubelet https://kubernetes.io/docs/reference/generated/kubelet/
After=docker.service
Requires=docker.service

[Service]
CPUAccounting=true
MemoryAccounting=true

WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \
	    $KUBE_LOGTOSTDERR \
	    $KUBE_LOG_LEVEL \
	    $KUBELET_KUBECONFIG \
	    $KUBELET_ADDRESS \
	    $KUBELET_PORT \
	    $KUBELET_HOSTNAME \
	    $KUBE_ALLOW_PRIV \
	    $KUBELET_ARGS
Restart=always
StartLimitInterval=0
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
```


### second node

```
# NOTE: need bootstrap config
# delete file /etc/kubernetes/kubelet.kubeconfig

KUBELET_KUBECONFIG="--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
KUBELET_ARGS="--config=/var/lib/kubelet/config.yaml --cgroup-driver=systemd --pod-infra-container-image=k8s.gcr.io/pause:3.1 --allow-privileged=true --network-plugin=cni --runtime-cgroups=/docker-daemon --kubelet-cgroups=/kubelet"
```


Remove node with name `home`:

```
kubectl drain home --delete-local-data --force --ignore-daemonsets && kubectl delete node home && kubeadm reset
```


## Operations

Run image:

```
# untaint master node (allowing scheduling to master)
kubectl taint node zz node-role.kubernetes.io/master:NoSchedule-

kubectl run -it some-pod --image=busybox --restart=Never /bin/sh

# To delete
kubectl delete pod some-pod


# Run on specific node
kubectl run -it pod4 --image=busybox --restart=Never --overrides='{ "apiVersion": "v1", "spec": { "template": { "spec": { "nodeSelector": { "kubernetes.io/hostname": "zz" } } } } }' /bin/sh'
```


```
kubectl get nodes -owide

kubectl describe nodes zz
```


## Troubleshooting


#### No network between pods on different hosts

Check `/etc/cni/net.d/` here should be file `10-flannel.conflist`

If you started any pod on this server you should have `cni0` interface in `ip a`

If not, you should:

* install `cni` and `cni-plugins`
* run `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml` or other string from flannel's documentation
* check that you have `--network-plugin=cni` in KUBELET_ARGS
* `sysctl net.bridge.bridge-nf-call-iptables=1`
