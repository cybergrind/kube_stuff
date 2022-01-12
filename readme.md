# 2021 edition

https://wiki.archlinux.org/title/Kubernetes
https://docs.projectcalico.org/getting-started/kubernetes/quickstart
https://docs.projectcalico.org/getting-started/clis/calicoctl/install
[cilium + kubeadm](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-kubeadm/)


#### Aliases:
```bash
# kubernetes
alias kp='k get po -A -o wide'
alias kev="kubectl get events --sort-by='.metadata.creationTimestamp' -A"
```

#### Install packages:
```
# control only
yay -S etcd kubernetes-control-plane
# helpers
yay -S kubectx  # kubectx kubens
# all
yay -S kubernetes-node kubeadm kubelet

# init
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sysctl --system


sudo systemctl enable kubelet
sudo systemctl start kubelet

```

Init kubernetes:
```
# kubeadm token generate and put into kubeadm-config.yaml:token

sudo kubeadm init --node-name=zz --config=kubeadm-config.yaml
# or sudo kubeadm init --pod-network-cidr='10.85.0.0/16' --node-name=zz
# kubeadm token create --print-join-command

# if cannot create, check kubelet parameters, they should be like:
# ExecStart=/usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.5
# + check cgroupDriver: cgroupfs OR systemd

# systemctl edit docker
[SERVICE]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// --data-root=/docker --exec-opt native.cgroupdriver=systemd

# comment /etc/kubernetes/kubelet.env
#KUBELET_ARGS=--cni-bin-dir=/usr/lib/cni


mkdir -p ~/.kube && sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
# allow running nodes on master
kubectl taint nodes --all node-role.kubernetes.io/master-

# see details below
kubectl apply -f calico.yaml
linkerd install | kubectl apply -f -
sudo calicoctl node status

# linkerd related
linkerd viz install | kubectl apply -f -
linkerd buoyant install | kubectl apply -f -
linkerd viz dashboard

```

Various:
```
# set namespace
kubens longhorn-system
# OR
kubectl config set-context --current --namespace longhorn-system

# get all resources in namespace
k get all -n longhorn-system

# for upgrade
sudo kubeadm upgrade apply 1.22.4
# https://blog.honosoft.com/2020/01/31/kubeadm-how-to-upgrade-update-your-configuration/
kubeadm config print init-defaults --component-configs KubeletConfiguration > kubeadm-config.yaml

# edit
kubeadm upgrade diff --config kubeadm-config.yaml
kubeadm upgrade apply --config kubeadm-config.yaml --ignore-preflight-errors all --force --v=5



curl https://docs.projectcalico.org/manifests/calico.yaml -O
# edit in calico.yml
# put your CIDR and autodetect method

## - name: CALICO_IPV4POOL_CIDR
##   value: "10.85.0.0/16"
## - name: IP_AUTODETECTION_METHOD
##   value: "can-reach=8.8.8.8"


# not working
## # download and edit https://docs.projectcalico.org/manifests/custom-resources.yaml
## kubectl create -f custom-resources.yaml

# after this it will be created /etc/cni/net.d



# calicoctl
cd ~/.local/bin
curl -o calicoctl -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.21.0/calicoctl" && chmod +x calicoctl

curl -o kubectl-calico -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.21.0/calicoctl" && chmod +x kubectl-calico

# check
calicoctl get nodes -o wide

# linkerd
# curl -fsL https://run.linkerd.io/install | sh
yay -Ss linkerd
linkerd check --pre
linkerd install | kubectl apply -f -

linkerd check

# demo app
curl -fsL https://run.linkerd.io/emojivoto.yml | kubectl apply -f -
kubectl -n emojivoto port-forward svc/web-svc 8080:80

# delete demo
curl -fsL https://run.linkerd.io/emojivoto.yml | kubectl delete -f -


# inject linkerd
kubectl get -n emojivoto deploy -o yaml | linkerd inject -| kubectl apply -f -


# viz
linkerd viz install | kubectl apply -f -
linkerd buoyant install | kubectl apply -f -

# if failed you can get config and apply it from dashboard
# https://buoyant.cloud/agent/buoyant-cloud-k8s-zz-8=.yml | kubectl apply -f -

linkerd viz dashboard

## DELETE node
kubectl drain tpad --force --ignore-daemonsets --delete-emptydir-data
sudo kubeadm reset

kubectl uncordon tpad

#taint back / Adding a taint to an existing node using NoSchedule
kubectl untaint nodes node1 dedicated=special-user:NoSchedule

```

#### Traefik:

```
helm repo add traefik https://helm.traefik.io/traefik
helm repo update
helm install traefik traefik/traefik


# update things
helm show values traefik/traefik > traefik.values.yaml
helm upgrade -f traefik.values.yaml traefik traefik/traefik
# or apply linkerd
helm template -f traefik.values.yaml traefik traefik/traefik | linkerd inject - | kubectl apply -f -

traefik apply -f traefik_dashboard.yaml

# check external ip
kubectl get svc
kubectl describe svc traefik
```

Basic auth: check [traefik/traefik_dashboard_remote.yaml](/traefik/traefik_dashboard_remote.yaml)


#### Registry

```
docker run -d -p 5005:5000 --restart=always --name registry registry:2

# edit docker service
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock  --insecure-registry=10.0.0.111:5005

# use images in format: 10.0.0.111:5005/imagename

```
check https://github.com/cybergrind/fastapi_and_vue/blob/main/infra/helm-chart/templates/registry.yaml


#### openkruise

https://openkruise.io/docs/installation
```
helm repo add openkruise https://openkruise.github.io/charts/
helm install kruise openkruise/kruise --version 1.0.0

```

#### Calico Commands:

```

kubectl get pods -n calico-system
kubectl get pods -n linkerd

kubectl get nodes -owide
kubectl describe nodes zz

# redeploy
kubectl -n {NAMESPACE} rollout restart deploy

# get deploy
kubectl get -n emojivoto deploy -o yaml
```

Default troubleshooting way:

```
kubectl get namespaces
kubectl get pods -n NS
kubectl describe pod POD

calicoctl get nodes -o wide
kubectl get -n emojivoto deploy -o yaml | linkerd inject -| kubectl apply -f -


# viz
linkerd viz install | kubectl apply -f -
linkerd buoyant install | kubectl apply -f -

# if failed you can get config and apply it from dashboard
# https://buoyant.cloud/agent/buoyant-cloud-k8s-zz-8=.yml | kubectl apply -f -

linkerd viz dashboard

## DELETE node
kubectl drain tpad --force --ignore-daemonsets --delete-emptydir-data
sudo kubeadm reset
sudo systemctl stop kubelet
sudo rm -rf /etc/cni/net.d /etc/kubernetes

kubectl uncordon tpad
```

Other Commands:

```
# redeploy
kubectl -n {NAMESPACE} rollout restart deploy

# get deploy yaml for edit
kubectl get -n emojivoto deploy -o yaml


# debug
kubectl debug -it --image ghcr.io/micro-fan/python:4.0.4 vote-bot-6bd795dbc-hm4cn -c dd -n emojivoto --share-processes -- /bin/bash
kubectl attach ddd -c dd -i -t

# edit config
kubectl edit pod -n emojivoto vote-bot

# restart
kubectl scale -n emojivoto deployment --replicas 0 vote-bot
kubectl scale -n emojivoto deployment --replicas 1 vote-bot
```

Default troubleshooting way:

```
kubectl get namespaces
kubectl get pods -n NS
kubectl describe pod POD

calicoctl get nodes -o wide
sudo calicoctl get node status
```

zookeper + linkerd
```
helm repo add bitnami https://charts.bitnami.com/bitnami
kubectl create ns dev
helm upgrade --install zookeeper bitnami/zookeeper --namespace dev --set replicaCount=3 --set podAnnotations.'linkerd\.io\/inject'=enabled
```

# 2019 edition

## flannel (local installation for docker)

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


## kubernetes

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
