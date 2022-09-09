# k3s (lightweight kubernetes) on Synology DS216+

## "demo"

```
root@synology:~# kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   metrics-server-6d684c7b5-hnskg            1/1     Running   0          10h
kube-system   local-path-provisioner-58fb86bdfd-qrb7n   1/1     Running   0          10h
kube-system   coredns-5b66977746-d9k9c                  1/1     Running   0          10h
kube-system   traefik-7b8b884c8-jppf4                   1/1     Running   0          10h
kube-system   svclb-traefik-n84wx                       2/2     Running   0          10h

root@synology:~# uname -a
Linux backup 3.10.108 #42218 SMP Tue Apr 26 04:09:35 CST 2022 x86_64 GNU/Linux synology_braswell_216+
```

## notes about DSM version

* 6.2.x - it would require rebuild of iptables to update it from 1.6.0 to something above/equal to 1.8.1
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
root@laptop:~# mkdir -p /synology/tarballs
root@laptop:~# cd /synology/tarballs
root@laptop:/synology/tarballs# wget https://global.download.synology.com/download/ToolChain/toolkit/7.0/base/base_env-7.0.txz
root@laptop:/synology/tarballs# wget https://global.download.synology.com/download/ToolChain/toolkit/7.0/braswell/ds.braswell-7.0.dev.txz
root@laptop:/synology/tarballs# wget https://global.download.synology.com/download/ToolChain/toolkit/7.0/braswell/ds.braswell-7.0.env.txz
root@laptop:/synology/tarballs# wget https://global.download.synology.com/download/ToolChain/Synology%20NAS%20GPL%20Source/7.0-41890/braswell/linux-3.10.x.txz
```

As you can see I'm using files for braswell arch (rather subarch) as this is what DS216+ is shipped with. If you have different model you have to use different files.

```
root@laptop:/synology/env# mkdir /synology/env
root@laptop:/synology/env# cd /synology/env
root@laptop:/synology/env# tar Jxvf ../tarballs/base_env-7.0.txz 
root@laptop:/synology/env# tar Jxvf ../tarballs/ds.braswell-7.0.dev.txz
root@laptop:/synology/env# tar Jxvf ../tarballs/ds.braswell-7.0.env.txz
root@laptop:/synology/env# tar Jxvf ../tarballs/linux-3.10.x.txz -C usr/local
```
### modules compilation

```
root@laptop:/synology/env# chroot 
CHROOT@ds.braswell[/]# cd /usr/local/linux-3.10.x/
CHROOT@ds.braswell[/usr/local/linux-3.10.x]# cp synoconfigs/braswell .config
CHROOT@ds.braswell[/usr/local/linux-3.10.x]# make menuconfig
```
We need these two options to be marked as module:

```
Symbol: IP_NF_TARGET_REJECT [=n]
Type  : tristate
Prompt: REJECT target support
  Location:
    -> Networking support (NET [=y]) 
      -> Networking options  
        -> Network packet filtering framework (Netfilter) (NETFILTER [=y]) 
          -> IP: Netfilter Configuration
            -> IP tables support (required for filtering/masq/NAT) (IP_NF_IPTABLES [=m])
              -> Packet filtering (IP_NF_FILTER [=m])
```

```
Symbol: NETFILTER_XT_MATCH_COMMENT [=n] 
Type  : tristate
Prompt: "comment" match support
  Location:
    -> Networking support (NET [=y])
      -> Networking options
        -> Network packet filtering framework (Netfilter) (NETFILTER [=y])
          -> Core Netfilter Configuration
            -> Netfilter Xtables support (required for ip_tables) (NETFILTER_XTABLES [=m])
```
Once you mark them as modules you can exit menuconfig and save changes.

Then we can compile modules:

```
CHROOT@ds.braswell[/usr/local/linux-3.10.x]# make modules
```

Once it's done this command should have same output:

```
CHROOT@ds.braswell[/usr/local/linux-3.10.x]# find | grep -E '(REJECT|comment).ko$'
./net/ipv4/netfilter/ipt_REJECT.ko
./net/netfilter/xt_comment.ko
```

Now you have to copy these two files to /lib/modules on your Synology.
Once that's done these two commands shouldn't return any errors:

```
root@synology:~# insmod /lib/modules/ipt_REJECT.ko 
root@synology:~# insmod /lib/modules/xt_comment.ko 
```

## k3s instalation

### notes on k3s / kubernetes versions

This tutorial is about installing version v1.17.17+k3s1.

And here goes explanation why.

Starting from 1.20.x k3s/kubernetes requires CGROUP PIDS which doesn't exist on kernel 3.10.x

Starting from 1.18.x k3s/kubernetes needs SECCOMP which in theory can be compiled in even on 3.10.x kernel but I didn't have time to work on that yet.

### actual installation

k3s has below option but in version 17.x it's pretty broken so we're gonna cheat it to install in /volume1 by creating symlink.

```
   --data-dir value, -d value                 (data) Folder to hold state default /var/lib/rancher/k3s or ${HOME}/.rancher/k3s if not root
```

```
root@synology:~# mkdir /volume1/rancher
root@synology:~# ln -s /volume1/rancher /var/lib/rancher
root@synology:~# curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.17.17+k3s1  sh -
```

At this point k3s should be running but it will stuck trying to create containers/pods. We have to fix one more thing so let's stop k3s for now.

```
root@synology:~# systemctl stop k3s
```

During startup k3s should create containerd configuration file and we have to make copy of it and modify it.

```
root@synology:~# cp /volume1/rancher/k3s/agent/etc/containerd/config.toml /volume1/rancher/k3s/agent/etc/containerd/config.toml.tmpl 
```

Now edit config.toml.tmpl and add the following section to it:

```
[plugins.cri.containerd]
 snapshotter = "native"
```
Now you can start k3s again:

```
root@synology:~# systemctl start k3s
```
And you're done!

### systemctl changes
To load modules on start/reboot you have to modify /etc/systemd/system/k3s.service and add the following two lines before ExecStart command:

```
ExecStartPre=-/sbin/insmod /lib/modules/ipt_REJECT.ko
ExecStartPre=-/sbin/insmod /lib/modules/xt_comment.ko
```

And run systemctl daemon-reload afterwards.

# kudos / thanks

I would lie saying that I haven't used https://medium.com/@marco.mezzaro/k3s-on-synology-what-if-it-works-e980b4b09fcb / https://github.com/marcomezzaro/synok3s as a starting points for my own work but given how many modifications I had to make I'd rather say that Marco's work was just inspiration. But nevertheless his work was very useful and kudos for him!
