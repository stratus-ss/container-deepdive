## Introduction

Today we are going to take a look at the `mount` namespace. This is the third in the Linux Namespace series. In the [first article](https://www.redhat.com/sysadmin/7-linux-namespaces) I gave an introduction to the most used namespaces, laying the groundwork for for the hands on work we started in the [user namespaces]() article. I’m hoping to build out some fundamental knowledge as to how the underpinnings of Linux Containers work. If you are interested in how Linux controls the resources on a system, check out the [CGroup series](https://www.redhat.com/sysadmin/cgroups-part-one) I wrote earlier. Hopefully by the time we are done with the namespaces hands on work, I can tie CGroups and Namespaces together in a meaningful way completing the picture.

For now, however, we are going to take a closer look at the `mount` namespace and how it can help us get closer to understanding the isolation that Linux containers bring us and by extension platforms like OpenShift or Kubernetes.


## The `Mount` Namespace

The `mount` namespace doesn't behave like you might expect after creating a new `user` namespace. By default if you were to create a new `mount` namespace with `unshare -m` your view of the system remains largely unchanged and unconfined. That is because whenever a new `mount` namespace is created, a _copy_ of the mount points from the parent namespace are created in the newly created `mount` namespace. That means that any action taken on files inside a poorly configured `mount` namespace **will** impact the host. 

### Some Setup Steps For `Mount` Namespaces

So what use is the `mount` namespace then? To help demonstrate this, I will be using a tarball of [Alpine Linux](https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86_64/alpine-minirootfs-3.13.1-x86_64.tar.gz) as Alpine is one of the most popular bases for container images.

Just briefly then, I downloaded it, untarred it and moved it into a new directory, giving the top level directory permissions for my unprivileged user:

```
[root@localhost ~] export CONTAINER_ROOT_FOLDER=/container_practice
[root@localhost ~] mkdir -p ${CONTAINER_ROOT_FOLDER}/fakeroot
[root@localhost ~] cd ${CONTAINER_ROOT_FOLDER}
[root@localhost ~] wget https://dl-cdn.alpinelinux.org/alpine/v3.13/releases/x86_64/alpine-minirootfs-3.13.1-x86_64.tar.gz
[root@localhost ~] tar xvf alpine-minirootfs-3.13.1-x86_64.tar.gz -C fakeroot
[root@localhost ~] chown container-user. -R ${CONTAINER_ROOT_FOLDER}/fakeroot
```

The reason the `fakeroot` directory needs to be owned by the user `container-user` is because once we create a new `user` namespace, the `root` user in the new namespace will be mapped to the `container-user` outside of the namespace. This means that processes inside of the new namespace will think that it has the capabilities required to modify its own files, but the file system permissions of the host will prevent the `container-user` from modifying the Alpine files from the tarball (which have `root` as the owner).

So what happens if you simply start a new `mount` namespace? Let's take a look:
```
PS1='\u@new-mnt$ ' unshare -Umr
```

Now that we are inside our new namespace, you might expect to not be able to see any of the original mount points from the host. However, this is not the case:

```
root@new-mnt$ df -h
Filesystem           Size  Used Avail Use% Mounted on
/dev/mapper/cs-root   36G  5.2G   31G  15% /
tmpfs                737M     0  737M   0% /sys/fs/cgroup
devtmpfs             720M     0  720M   0% /dev
tmpfs                737M     0  737M   0% /dev/shm
tmpfs                737M  8.6M  728M   2% /run
tmpfs                148M     0  148M   0% /run/user/0
/dev/vda1            976M  197M  713M  22% /boot


root@new-mnt$ ls /
bin   container_practice  etc   lib    media  opt   root  sbin  sys  usr
boot  dev                 home  lib64  mnt    proc  run   srv   tmp  var
```

The reason for this is that `systemd` defaults to recursively sharing the mount points with all new namespaces. If we went ahead and mounted a `tmpfs` filesystem somewhere, for example `/mnt` inside of the new `mount` namespace, is the host able to see it?

```
root@new-mnt$ mount -t tmpfs tmpfs /mnt

root@new-mnt$ findmnt |grep mnt
└─/mnt     tmpfs               tmpfs      rw,relatime,seclabel,uid=1000,gid=1000
```

The host however does not see this:

```
[root@localhost ~]# findmnt |grep mnt
```

So at the very least we know that the `mount` namespace is functioning correctly. This is a good time to take a small detour to discuss propagation of mount points. I'm going to summarize just briefly but if you are interested in a greater understanding have a look at Michael Kerrisk's [LWN article](https://lwn.net/Articles/689856/) as well as the [man page for the mount namespace](https://www.man7.org/linux/man-pages/man7/mount_namespaces.7.html). I don't normally rely so much on the man pages as I often find that they are not easily digestible. However, in this case they are full of examples and (mostly) plain English.

### Theory of Mountpoints

Mounts propagate by default because of a feature in the kernel called the **shared subtree**. This allows every mount point to have its own propagation type associated with it. This metadata determines whether new mounts under a given path are propagated to other mount points. The example given in the man page is that of an optical disk. If your optical disk automatically mounted under `/cdrom`, the contents would only be visible in other namespaces if the appropriate propagation type is set.

#### Peer Groups & Mount States

The [kernel documentation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt) says that a "'*peer group*' is defined as a group of vfsmounts that propagate events to each other." Events are things such mounting a network share or unmounting an optical device. Why is this important you may ask? Well, when it comes to the `mount` namespace *peer groups* are often the deciding factor as to whether or not a mount is visible and can be interacted with. A **mount state** determines whether or not a member in a *peer group* can receive the event. According to the same kernel documentation, there are 5 mount states:

1) **shared** - A shared mount is a mount that belongs to a *peer group*. Any changes that occur will propagate through all members of the *peer group*
2) **slave** - One way propagation. The 'master' mount point will propagate events to a slave but any actions the slave takes will not been seen by the master
3) **shared and slave** - This state indicates that the mount point has a 'master' but it also has its own *peer group*. The 'master' will not be notified of changes to a mount point, but any *peer group* members downstream will.
4) **private** - This point point does not receive or forward any propagation events.
5) **unbindable** - This point point does not receive or forward any propagation events and CANNOT be bind mounted.

It's important to note that the mount point state is *per mount point*. That means if you have `/` and `/boot` for instance, you would have to apply the desired state to each mount point separately. 

In case you are wondering, in the case of containers, most engines (docker, crio etc) use private mount states when mounting a volume inside a container.

We aren't going to worry too much about this for now. I just wanted to provide some context. If you want to try some specific mounting scenarios take a look at the [man pages](https://www.man7.org/linux/man-pages/man7/mount_namespaces.7.html) as the examples are quite good.

### Creating Our Mount Namespace

If we were using a programming language like `Go` or `C` we could use the raw system kernel calls to create the appropriate environment for our new namespace(s). However, since the intent behind this is to help you understand how to interact with a container that already exists, we will have to do some bash trickery to get our new `mount` namespace into the desired state. 

First off, we'll start by by creating our new `mount` namespace as a regular user:

```
unshare -Urm
```

Once we are inside the namespace let's take a look at the `findmnt` of the mapper device which contains the root file system (for brevity I have removed most of the mount options from the output):

```
findmnt |grep mapper

/       /dev/mapper/cs-root      xfs           rw,relatime,[...]
```

We can see that there is only one mount point that has the root device mapper. This is important because one of the things we are going to have to do is bind the mapper device into our Alpine directory:

```
export CONTAINER_ROOT_FOLDER=/container_practice
mount --bind ${CONTAINER_ROOT_FOLDER}/fakeroot ${CONTAINER_ROOT_FOLDER}/fakeroot
cd ${CONTAINER_ROOT_FOLDER}/fakeroot
```

The reason for this is because we are going to be using a utility called `pivot_root` in order to perform a `chroot`-like action. `pivot_root` takes two arguments `new_root` and `old_root` (sometimes referred to as `put_old`). `pivot_root` moves the root file system of the current process to the directory `put_old` and makes `new_root` the new root file system.

**IMPORTANT**: A note about `chroot`. `chroot` is often thought of as having extra security benefits. To some extent this is true as it takes a more significant amount of expertise to break free. A carefully constructed `chroot` can be very secure. However, `chroot` does not modify or restrict linux capabilities which we touched on in part 1. Nor does it restrict system calls to the kernel. This means that a sufficiently skilled aggressor could potentially escape a `chroot` that has not been thought through. The `mount` and `user` namespaces help to solve this problem.

If we try to use `pivot_root` without the bind mount, the command will say:

```
pivot_root: failed to change root from `.' to `old_root/': Invalid argument
```

In order to switch to the Alpine root filesystem, we'll first make a directory for `old_root` and then pivot into the intended (Alpine) root filesystem. Since Alpine Linux root filesystem does not have symlinks for `/bin` and `/sbin` we'll have to add those to our path and then finally, unmount the `old_root`:

```
mkdir old_root
pivot_root . old_root
PATH=/bin:/sbin:$PATH
umount -l /old_root
```

At this point we now have a nice environment where the `user` and `mount` namespaces are working together to provide a layer of isolation from the host. You can see that we no longer have access to binaries on the host. Try issuing the `findmnt` command that we have been using:

```
root@new-mnt$ findmnt
-bash: findmnt: command not found
```

You can also look at the root filesystem or attempt to see what is mounted:

```
root@new-mnt$ ls -l /
total 12
drwxr-xr-x    2 root     root          4096 Jan 28 21:51 bin
drwxr-xr-x    2 root     root            18 Feb 17 22:53 dev
drwxr-xr-x   15 root     root          4096 Jan 28 21:51 etc
drwxr-xr-x    2 root     root             6 Jan 28 21:51 home
drwxr-xr-x    7 root     root           247 Jan 28 21:51 lib
drwxr-xr-x    5 root     root            44 Jan 28 21:51 media
drwxr-xr-x    2 root     root             6 Jan 28 21:51 mnt
drwxrwxr-x    2 root     root             6 Feb 17 23:09 old_root
drwxr-xr-x    2 root     root             6 Jan 28 21:51 opt
drwxr-xr-x    2 root     root             6 Jan 28 21:51 proc
drwxr-xr-x    2 root     root             6 Feb 17 22:53 put_old
drwx------    2 root     root            27 Feb 17 22:53 root
drwxr-xr-x    2 root     root             6 Jan 28 21:51 run
drwxr-xr-x    2 root     root          4096 Jan 28 21:51 sbin
drwxr-xr-x    2 root     root             6 Jan 28 21:51 srv
drwxr-xr-x    2 root     root             6 Jan 28 21:51 sys
drwxrwxrwt    2 root     root             6 Feb 19 16:38 tmp
drwxr-xr-x    7 root     root            66 Jan 28 21:51 usr
drwxr-xr-x   12 root     root           137 Jan 28 21:51 var


root@new-mnt$ mount
mount: no /proc/mounts
```

Interestingly, there is no proc filesystem mounted by default. Let's try and mount it:

```
root@new-mnt$ mount -t proc proc /proc
mount: permission denied (are you root?)

root@new-mnt$ whoami
root
```

Because `proc` is a special type of mount related to the `pid` namespace we cannot mount it even though we are in our own `mount` namespace. This goes back to the capability inheritance that we talked about earlier. We'll pick this up discussion in the next article when we start discussing the `pid` namespace. However, as a reminder about inheritance, have a look at the diagram below:

![mount_namespace.png](/namespaces/mount_namespace.png)

In the next article we'll rehash this diagram but if you have been following along since the beginning, you should be able to make some inferences before then. 

## Wrapping Up

In this article we took a look at some deeper theory around the `mount` namespace. We also talked about *peer groups* and how they relate to the mount states that are applied to each mount point on a system. For the hands on part, we downloaded a minimal Alpine Linux file system and then walked through how to use the `user` and `mount` namespaces to create an environment that looks a lot like `chroot` except potentially more secure.

For now, go ahead and play around with mounting file systems inside and outside of your new namespace. Try creating new mount points that make use of the `shared`, `private` and `slave` mount states. 





