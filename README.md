# k3s (lightweight kubernetes) on Synology DS216+

```
root@backup:~# kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   metrics-server-6d684c7b5-hnskg            1/1     Running   0          10h
kube-system   local-path-provisioner-58fb86bdfd-qrb7n   1/1     Running   0          10h
kube-system   coredns-5b66977746-d9k9c                  1/1     Running   0          10h
kube-system   traefik-7b8b884c8-jppf4                   1/1     Running   0          10h
kube-system   svclb-traefik-n84wx                       2/2     Running   0          10h
root@backup:~# uname -a
Linux backup 3.10.108 #42218 SMP Tue Apr 26 04:09:35 CST 2022 x86_64 GNU/Linux synology_braswell_216+
```

# notes about DSM version
* 6.2.x - it would require rebuild of iptables to update it from 1.6.0 to something above 1.8.1
* 7.1.x - might work at some point when Synology release kernel/toolchain/toolkit for this version
* 7.0.x - WORKS and this tutorial is about how to make it

Steps to create working k3s setup.

Kernel.
