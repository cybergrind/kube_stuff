
#### ENI pod limit

https://aws.amazon.com/blogs/containers/amazon-vpc-cni-increases-pods-per-node-limits/
```bash

# check
kubectl describe daemonset -n kube-system aws-node | grep ENABLE_PREFIX_DELEGATION

# update
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# faster allocation
kubectl set env ds aws-node -n kube-system WARM_PREFIX_TARGET=1

```
