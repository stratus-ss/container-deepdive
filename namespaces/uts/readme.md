# Intro

This namespace is often mis-understood by the casual observer, largely owing to the fact that the name no longer matches its purpose. UTS actually stands for the Unix Timesharing System. However, this namespace actually controls the hostname and the NIS domain. It is important to note that the Man Page describes the UTS namespace as follows: 

>   These identifiers are set using sethostname(2) and setdomainname(2), and can be retrieved using uname(2), gethostname(2), and getdomainname(2). Changes made to these identifiers are visible to all other processes in the same UTS namespace, but are not visible to processes in other UTS namespaces. 
{.is-info}

This means that some of the more modern tooling (*systemd* and others) may not cause the changes we expect. We'll touch more on this later.

As you can imagine, there might be several use cases where you might want processes to have different hostnames. Web Servers for example, tend to throw a warning if the hostname does not match the SSL certificate they are serving. On the other hand, some processes might attach the hostname to a network process. An incorrect hostname can cause failed or denied connections or a myriad of other problems.

With all that said, let's get into some examples.

## Exploring The UTS Namespace


You can invoke the namespace by doing

```
unshare --uts /bin/bash
```

However, one thing that you will notice is that once you are in the new namespace, using `hostnamectl set-hostname` will not change the hostname in the new namespace.

```
[root@bastion ~]# unshare --uts
[root@bastion ~]# hostname
bastion.stratus.lab
[root@bastion ~]# hostnamectl set-hostname tux
[root@bastion ~]# hostname
bastion.stratus.lab
```

However if you open a new shell you will notice that the hostname has actually changed:

```
[user@host ~]$ ssh root@bastion
Last login: Tue Dec  7 08:17:48 2021 from 192.168.99.198
[root@tux ~]# hostname
tux
```

Why is this? *systemd* does not execute the `sethostname` system call. Instead, *systemd* completes the task by connecting to a socket. Since the socket is associated with the old namespace, the old namespace hostname is adjusted but not the new namespace.

### Applying Lessons Of The Past

#### Using Root Namespaces

Let's do a brief review of the [Mount Namespace](https://www.redhat.com/sysadmin/mount-namespaces) article. 
In it, the following note appears:

> Now that you're inside the new namespace, you might not expect to see any of the original mount points from the host. However, this isn't the case. The reason for this is that *systemd* defaults to recursively sharing the mount points with all new namespaces. 
{.is-info}

*systemd* derives a lot of its information from `/run` which is shared into the namespace we have created here. In the Mount Namespace article, we mounted `tmpfs` into the new namespace on a directory we did not want to share with the old namespace.

You can disable the most of *systemd* calls by mounting the new namespace with the following options:

```
unshare --mount --uts /bin/bash
```

Then remount `/run`:

```
mount -t tmpfs tmpfs /run
```

The whole process will look like this:

```
[root@bastion ~]# unshare --fork --mount --uts /bin/bash
[root@bastion ~]# mount -t tmpfs tmpfs /run
[root@bastion ~]# hostnamectl set-hostname bastion.stratus.lab
System has not been booted with systemd as init system (PID 1). Can't operate.
Failed to connect to bus: Host is down
[root@bastion ~]# hostname tux
[root@bastion ~]# hostname
tux
```

You can confirm this by opening a new shell to the box:

```
[user@host ~]$ ssh root@bastion
Last login: Tue Dec  7 08:33:04 2021 from 192.168.99.198
[root@bastion ~]# hostname
bastion.stratus.lab
```

If you remember from the previous articles, you can find the appropriate namespace by using the `lsns` command:

```
[root@bastion ~]# lsns |grep uts
4026531838 uts       133     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 31
4026532250 uts         1   645 root   /usr/lib/systemd/systemd-udevd
4026532405 uts         1   743 root   /usr/lib/systemd/systemd-logind
4026532479 mnt         2 11507 root   unshare --fork --mount --uts /bin/bash
4026532480 uts         2 11507 root   unshare --fork --mount --uts /bin/bash
```

You can then enter the namespace with the `nsenter` command:

```
[root@bastion ~]# nsenter -t 11507 -a 
[root@tux /]#
```

#### Using A User Namespace

Recall from the [User Namespace](https://www.redhat.com/sysadmin/building-container-namespaces) article that there are some additional considerations when creating namespaces as an unprivileged user.

> When you create a new user namespace, your current user will be mapped to the user **nobody**. This is because, by default, there is no user ID mapping taking place. When no mapping is defined, the namespace simply uses your system's rules to determine how to handle an undefined user.
{.is-info}

Refer back to the article for more on user mapping. In this case, we will want to have the **root** user mapped for us automatically. This way we will have "root" in the new namespace. (Again see the first article for the discussion on namespaces and permissions).

If we put our combine lessons into practice on a CentOS Stream 9 host, we can observe the following:

```
ocp@bastion ~  $ unshare --map-root-user --user --mount --uts --fork /bin/bash
root@bastion ~  $ hostnamectl set-hostname tux
Could not set static hostname: Interactive authentication required.
```

This is because of how `polkit` is configured on the RHEL family of distributions. Other distributions do not necessarily throw this error. Arch Linux (with no special `polkit` configuration), for example, still allows the container to change the hostname. Therefore regardless of your distribution, it is still good security practice to remount `/run`.

> It is important to note that the output of `lsns` might be more easily read if you use `--fork` when creating the namespaces. 
> 
> **Without `--fork`**
> 
> ```
> [root@bastion ~]# lsns |grep uts
> 4026531838 uts       135     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 31
> 4026532250 uts         1   645 root   /usr/lib/systemd/systemd-udevd
> 4026532405 uts         1   743 root   /usr/lib/systemd/systemd-logind
> 4026532414 uts         1 11962 ocp    /bin/bash
> ```
> 
> **With `--fork`**
> 
> ```
> [root@bastion ~]# lsns |grep uts
> 4026531838 uts       134     1 root   /usr/lib/systemd/systemd --switched-root --system --deserialize 31
> 4026532250 uts         1   645 root   /usr/lib/systemd/systemd-udevd
> 4026532405 uts         1   743 root   /usr/lib/systemd/systemd-logind
> 4026532412 user        2 11939 ocp    unshare --map-root-user --user --mount --uts --fork /bin/bash
> ```
> 
> While `--fork` is not strictly necessary in this scenario, it may be useful or even required if a new PID namespace is required (see [PID Namespace](https://www.redhat.com/sysadmin/pid-namespace) article for more information)
> 
{.is-warning}


# Wrapping Up

The UTS namespace is not the most complicated Linux namespace. It is, however, quite useful. Especially in the context of containers. To make the most of the UTS namespace it should be combined with the `mount` namespace at a minimum when using the root namespace and `mount` and `user` namespaces when spawning from an unprivileged user.

In the next article we will discuss the `net` namespace and how it can be used to enable namespaces to have their own ip and port space.









