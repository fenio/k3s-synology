# k3s (lightweight kubernetes) on Synology DS216+

## "demo"

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

## notes about DSM version

* 6.2.x - it would require rebuild of iptables to update it from 1.6.0 to something above/equal 1.8.1
* 7.1.x - might work at some point when Synology releases kernel/toolchain/toolkit for this version
* 7.0.x - WORKS and this tutorial is all about it

If you're already on DSM 7.1.x it's possible to do downgrade despite Synology saying it's not possible.
Everything is possible... after all it's Linux :P
Just google for it.

## additional kernel modules for iptables

DSM 7.0.x comes with iptables 1.8.3 but not all kernel modules are available.
We care only about ipt_REJECT and xt_comment. Without them k3s fails.

### Creating environment for building kernel modules

In theory you could use https://github.com/SynologyOpenSource/pkgscripts-ng to create toolchain to compile kernel modules but despite having "ng" in its name it's rather old and unmaintained piece of software. Just forget about it. Well for 6.2.x it's a good starting point but we're working on 7.0.x and for anything newer than 6.2.x their scripts simply don't work.

But happily these scripts don't do any magic. Based on what happens for 6.2.x I was able to prepare newer env myself with simple commands.
We need some working Linux system. I personally use Debian but it shouldn't really matter. Use whatever you prefer. Just don't assume it's happening on NAS.


```
mkdir -p /synology/tarballs
cd /synology/tarballs
wget https://global.download.synology.com/download/ToolChain/toolkit/7.0/base/base_env-7.0.txz
wget https://global.download.synology.com/download/ToolChain/toolkit/7.0/braswell/ds.braswell-7.0.dev.txz
wget https://global.download.synology.com/download/ToolChain/toolkit/7.0/braswell/ds.braswell-7.0.env.txz
wget https://global.download.synology.com/download/ToolChain/Synology%20NAS%20GPL%20Source/7.0-41890/braswell/linux-3.10.x.txz
```

As you can see I'm using files for braswell arch (rather subarch) as this is what DS216+ is shipped with. If you have different model you have to use different files.

