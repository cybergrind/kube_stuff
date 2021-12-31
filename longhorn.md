
```bash
yay -S open-iscsi

cat /etc/kubernetes/kubelet.env
KUBELET_ARGS="--allow-privileged=true"


k apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.2.3/deploy/longhorn.yaml
k get po -n longhorn-system --watch
```
