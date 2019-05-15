# Namespace-Lister

List used namespace on a linux system. (All credit to [Ralf Trezeciak](https://www.opencloudblog.com/?p=251))

**Usage:** `./listns.py` or `python2 listns.py` 

---

# Mirror of the article

Namespaces in Linux are heavily used by many applications, e.g. LXC, Docker and Openstack.
Question: How to find all existing namespaces in a Linux system?

The answer is quite difficult, because it’s easy to hide a namespace or more exactly make it difficult to find them.

**Exploring the system**

In the basic/default setup Ubuntu 12.04 and higher provide namespaces for

- ipc for IPC objects and POSIX message queues
- mnt for filesystem mountpoints
- net for network abstraction (VRF)
- pid to provide a separated, isolated process ID number space
- uts to isolate two system identifiers — nodename and domainname – to be used by uname

These namespaces are shown for every process in the system. if you execute as root

```
ls -lai /proc/1/ns
 
60073292 dr-x--x--x 2 root root 0 Dec 15 18:23 .
   10395 dr-xr-xr-x 9 root root 0 Dec  4 11:07 ..
60073293 lrwxrwxrwx 1 root root 0 Dec 15 18:23 ipc -> ipc:[4026531839]
60073294 lrwxrwxrwx 1 root root 0 Dec 15 18:23 mnt -> mnt:[4026531840]
60073295 lrwxrwxrwx 1 root root 0 Dec 15 18:23 net -> net:[4026531968]
60073296 lrwxrwxrwx 1 root root 0 Dec 15 18:23 pid -> pid:[4026531836]
60073297 lrwxrwxrwx 1 root root 0 Dec 15 18:23 uts -> uts:[4026531838]
```

You get the list of attached namespaces of the init process using PID=1. Even this process has attached namespaces. These are the default namespaces for ipc, mnt, net, pid and uts. For example, the default net namespace is using the ID net:[4026531968]. The number in the brackets is a inode number.

In order to find other namespaces with attached processes in the system, we use these entries of the PID=1 as a reference. Any process or thread in the system, which has not the same namespace ID as PID=1 is not belonging to the DEFAULT namespace.

Additionally, you find the namespaces created by „ip netns add <NAME>“ by default in /var/run/netns/ .
  
**The python code**

The python code below is listing all non default namespaces in a system. The program flow is

- Get the reference namespaces from the init process (PID=1). Assumption: PID=1 is assigned to the default namespaces supported by the system
- Loop through /var/run/netns/ and add the entries to the list
- Loop through /proc/ over all PIDs and look for entries in /proc/<PID>/ns/ which are not the same as for PID=1 and add then to the list
- Print the result
  
The script listns.py is available on this repo, run it as root using `python2 listns.py`

```
       PID  Namespace             Thread/Command
        --  net:[4026533172]      created by ip netns add qrouter-c33ffc14-dbc2-4730-b787-4747
        --  net:[4026533112]      created by ip netns add qrouter-5a691ed3-f6d3-4346-891a-3b59
       297  mnt:[4026531856]      kdevtmpfs 
      3429  net:[4026533050]**    dnsmasq --no-hosts --no-resolv --strict-order --bind-interfa
      3429  mnt:[4026533108]      dnsmasq --no-hosts --no-resolv --strict-order --bind-interfa
      3486  net:[4026533050]**    /usr/bin/python /usr/bin/neutron-ns-metadata-proxy --pid_fil
      3486  mnt:[4026533107]      /usr/bin/python /usr/bin/neutron-ns-metadata-proxy --pid_fil
```

The example above is from an Openstack network node. The first four entries are entries created using the command ip. The entry PID=297 is a kernel thread and no user process. All other processes listed, are started by Openstack agents. These process are using network and mount namespaces. PID entries marked with `‚**‘` have a corresponding entry created with the ip command.

When a docker command is started, the output is:

```
       PID  Namespace             Thread/Command
        --  net:[4026532676]      created by ip netns add test
        35  mnt:[4026531856]      kdevtmpfs 
      6189  net:[4026532585]      [docker] /bin/bash 
      6189  uts:[4026532581]      [docker] /bin/bash 
      6189  ipc:[4026532582]      [docker] /bin/bash 
      6189  pid:[4026532583]      [docker] /bin/bash 
      6189  mnt:[4026532580]      [docker] /bin/bash 
```

The docker child running in the namespaces is marked using [docker].

On a node running mininet and a simple network setup the output looks like:

```
       PID  Namespace             Thread/Command
        14  mnt:[4026531856]      kdevtmpfs 
      1198  net:[4026532150]      bash -ms mininet:h1 
      1199  net:[4026532201]      bash -ms mininet:h2 
      1202  net:[4026532252]      bash -ms mininet:h3 
      1203  net:[4026532303]      bash -ms mininet:h4
```

**Googles Chrome Browser**

Googles Chrome Browser makes extensive use of the linux namespaces. Start Chrome and run the python script. The output looks like:

```
       PID  Namespace             Thread/Command
        63  mnt:[4026531856]      kdevtmpfs 
     30747  net:[4026532344]      /opt/google/chrome/chrome --type=zygote 
     30747  pid:[4026532337]      /opt/google/chrome/chrome --type=zygote 
     30753  net:[4026532344]      /opt/google/chrome/nacl_helper 
     30753  pid:[4026532337]      /opt/google/chrome/nacl_helper 
     30754  net:[4026532344]      /opt/google/chrome/chrome --type=zygote 
     30754  pid:[4026532337]      /opt/google/chrome/chrome --type=zygote 
     30801  net:[4026532344]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30801  pid:[4026532337]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30807  net:[4026532344]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30820  net:[4026532344]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30820  pid:[4026532337]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30887  net:[4026532344]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30887  pid:[4026532337]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30893  net:[4026532344]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30893  pid:[4026532337]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30901  net:[4026532344]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30901  pid:[4026532337]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30910  net:[4026532344]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30910  pid:[4026532337]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     30915  net:[4026532344]      /opt/google/chrome/chrome --type=renderer --lang=en-US --for
     
```
Chrome makes use of pid and network namespaces to restrict the access of subcomponents. The network namespace does not have a link in /var/run/netns/.

**Conclusion**

It’s quite hard to explore the Linux namespace. There is a lot of documentation flowing around. I did not find any simple program to look for namespaces in the system. So I wrote one.

The script cannot find a network namespace, which do not have any process attached to AND which has no reference in /var/run/netns/. If root creates the reference inode somewhere else in the filesystem, you may only detect network ports (ovs port, veth port on one side), which are not attached to a known network namespace –> an unknown guest might be on your system using a „hidden“ (not so easy to find) network namespace.

And — Linux namespaces can be stacked.
