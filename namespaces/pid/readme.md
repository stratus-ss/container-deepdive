


## Intro

Continuing on with our series on namespaces, this article will talk about the `PID` namespace. If you want a general overview of all the namespaces check out the [first article](https://www.redhat.com/sysadmin/7-linux-namespaces). When we last left off, we had walked through creating a new `mnt` namespace. Interestingly, as we discovered previously, even after creating a new mount namespace, we still had access to the original hosts's process IDs (PIDs). When we tried to mount the `/proc` namespace we greeted with the rather perplexing `permission denied` error as seen below:

```
root@new-mnt$ mount -t proc proc /proc
mount: permission denied (are you root?)

root@new-mnt$ whoami
root
```

While we were able to create all sorts of mounts in our new mount namespace, we couldn't interact or change `/proc`. In this article we'll step through the `PID` namespace and demonstrate how you can use it, along with the `mnt` namespace, to further secure our fledgeling container.



## What are Process IDs?

Before we jump straight into the `PID` namespace, I thought it would be a good idea to provide just a little bit of background as to why this namespace is important.

On most unix-like operating systems, when a process is created it is given a specific numeric identifier called a process ID. This helps to identify a unique process even if the task they are running is the same. For example if you had several processes running variations of the `sleep` command, you can use the process ID in order to stop the correct task.

All of these processes are tracked in a special file system called `procfs`. While this file system can technically be mounted anywhere, most tooling (and conventions) expect the `procfs` to be mounted under `/proc`. If you do a listing of `/proc` you will see a folder for every process currently running on your system. Inside this folder are all sorts of special files used for tracking various aspects of the process. For our purposes, these files are not important. It is sufficient to know that `/proc` is  the location that most unix-like systems store information regarding processes on a running system.


## The `PID` Namespace

One of the main reasons for the `PID` namespace is to allow for process isolation. More specifically, as the manpage says:
> PID namespaces isolate the process ID number space, meaning that processes in different PID namespaces can have the same PID.

This is important because it means that processes can be guaranteed not to have a conflicting PID with any other process. When considering a single system of course, there is no chance of PIDs conflicting because the system continuously increments the process ID number and never assigns the same number twice. However, when dealing with containers across multiple machines this issue becomes more salient. As described in the manpage:

> PID namespaces allow containers to provide functionality such as suspending/resuming the set of processes in the container and migrating the container to a new host while the processes inside the container maintain the same PIDs.

Aside from isolation, the PID system works *almost* identically to that outside of the namespace. The process IDs inside the new namespace start at `1` with the first process considered the _init_ process. The _init_ process is handled very differently from all the other PIDs on a host. This has special implications for the running system that are outside the scope of this series. If you are interested in more information see the "Signals and the init process" section of the [LWN Namespaces article](https://lwn.net/Articles/532748/).

What is of note however, is that whichever process has PID 1 is vital to the namespaces' longevity. If PID 1 is terminated for any reason, the kernel will send a `SIGKILL` to all remaining processes in the namespace, effectively shutting down that namespace.


### Exploring PID Namespaces

If you are wondering, like I was, about whether you can nest `PID` namespaces, the answer is yes. In fact, the kernel makes room for up to 32 nested `PID` namespaces. This is considered a one-way relationship. That means that the parent can see the pids of children, grandchildren etc. However, it cannot see any of the PIDs of its ancestors. Consider the following:


```
[user@localhost ~] sudo unshare -fp /bin/bash
[root@localhost ~] sleep 90000 &

[root@localhost ~] ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
[truncated ]
.....
root       11627   11620  0 09:16 pts/0    00:00:00 sudo unshare -fp /bin/bash
root       11633   11627  0 09:17 pts/0    00:00:00 unshare -fp /bin/bash
root       11634   11633  0 09:17 pts/0    00:00:00 /bin/bash
root       11639   11634  0 09:17 pts/0    00:00:00 sleep 90000
root       11641   11634  0 09:17 pts/0    00:00:00 ps -ef


[root@localhost ~] sudo unshare -fp /bin/bash
[root@localhost ~] sleep 8000 &

[root@localhost ~] ps -ef
[truncated ]
.....

UID          PID    PPID  C STIME TTY          TIME CMD
root       11650   11634  0 09:17 pts/0    00:00:00 sudo unshare -fp /bin/bash
root       11654   11650  0 09:17 pts/0    00:00:00 unshare -fp /bin/bash
root       11655   11654  0 09:17 pts/0    00:00:00 /bin/bash
root       11661   11655  0 09:17 pts/0    00:00:00 sleep 8000
root       11671   11655  0 09:17 pts/0    00:00:00 ps -ef

```

You will note that I truncated the output because the `PID` namespace *appears* to have full access to all of the PIDs in `/proc`. However if you attempt to stop a process that is in an ancestor what happens:

```
[root@localhost ~] kill -9 11361
bash: kill: (11361) - No such process
```

Why is this? Simply put, traditional tools like `ps` are not namespace aware and actually read from the `/proc` directory. If you did an `ls /proc` you would still see all of the folders and files from before because, as discussed in the last article, the `PID` namespace *inherits* all of the mounts from the `mnt` namespace. We'll address this situation later on. For now let's return to our example at hand.

In another shell, we then identify the sleeping processes:
```
[user@localhost ~] ps -ef |grep sleep
root       11639   11634  0 09:17 pts/0    00:00:00 sleep 90000
root       11661   11655  0 09:17 pts/0    00:00:00 sleep 8000
```

If you want to verify that these processes are in different namespaces, you will have to find the `bash` process PID. Remember, since we ran `sudo unshare -fp /bin/bash` the `bash` process is the _init_ process in the new namespace. Therefore, it is the PID that will be linked to the namespace ID. Let's grab the PIDs:

```
[root@localhost ~] ps -ef |grep bash
root       11627   11620  0 09:16 pts/0    00:00:00 sudo unshare -fp /bin/bash
root       11633   11627  0 09:17 pts/0    00:00:00 unshare -fp /bin/bash
root       11634   11633  0 09:17 pts/0    00:00:00 /bin/bash
root       11650   11634  0 09:17 pts/0    00:00:00 sudo unshare -fp /bin/bash
root       11654   11650  0 09:17 pts/0    00:00:00 unshare -fp /bin/bash
root       11655   11654  0 09:17 pts/0    00:00:00 /bin/bash
```

You can see the PIDs `11634` and `11655` in the output. If you compare this to the output of `lsns` (list namespaces), you will see the following:

```
[root@localhost ~] lsns |grep bash
        NS TYPE   NPROCS    PID USER             COMMAND
4026532952 pid         4 11634 root             /bin/bash
4026532954 pid         4 11655 root             /bin/bash
```

As you can see, the namespace IDs are different and thus the processes are in different namespaces.

Now that we have established the namespaces are indeed different, let's take a look at the PID ancestry mentioned before. You can do this by identifying the `NSpid` attribute of a given PID in the `/proc` directory as seen below:

```
sudo cat /proc/11655/status |grep NSpid
NSpid:	11655	6	1
```

The columns are read from left to right and indicate the PID in their respective namespaces. The left most PID is the primary or root namespace. In this case it has a PID of 11655, a secondary PID of 6 and a tertiary PID of 1. Since the namespaces own each descendant `PID` namespace, you can think of it like this:

On the host the `bash` process running the `sleep 8000` command has a PID of `11655`.
Inside the first "container" the `bash` process running the  `sleep 8000` command has a PID of `6`.
Inside the nested second "container" the PID is `1`. This is the container that actually launched the process.

Each one of these `bash` commands was created inside its own namespace but is visible to the parent (in this case the root namespace).

### `/proc`, PID And Unprivileged Users

The astute reader would have noticed that, while in the last two articles, a regular user was able to create both `user` and `mnt` namespaces, in this article I have been using the `sudo` command. This is because you are not able to create a `PID` namespace on its own with an unprivileged user. The answer to this is to combine multiple namespace creations into one event. There are a few different solutions to mounting `/proc` as an unprivileged user.

If we simply try create a new `user` namespace we will get a strange result:

```
[ user@localhost ~] unshare -Urp
-bash: fork: Cannot allocate memory
-bash-5.1# ps -ef
-bash: fork: Cannot allocate memory
-bash-5.1# ls
-bash: fork: Cannot allocate memory
```

What is happening? Remember how we said that the first process inside of a new PID namespace becomes the _init_ process? In this case, the current shell cannot move namespaces. It exists in the root namespace and when we created a new PID namespace the system did not know how to handle it. The solution to this is to have the process `fork` itself. This allows the current shell to become a child process of the `unshare` command. Using the `-f` flag results in the namespace being created:

```
[ user@localhost ~] unshare -Urp
[ root@localhost ~]
```

However, you are still seeing a contamination of the `/proc` mount point. There are two solutions to this. First you could create a new `mnt` namespace and then remount `/proc` yourself:

```
[ user@localhost  ~]$ unshare -Urpmf
[ root@localhost ~]# mount -t proc proc /proc
[ root@localhost ~]# ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 09:31 pts/0    00:00:00 -bash
root          10       1  0 09:31 pts/0    00:00:00 ps -ef
```

Indeed, for many years this was the only option. However, a flag `--mount-proc` was created some time ago to do this in a single step. The man page says:

> Just before running the program, mount the proc filesystem at mountpoint (default is /proc).  This is useful  when  creating a new PID namespace.  It also implies creating a new mount namespace since the /proc mount would otherwise mess up existing programs on the system.  The new proc filesystem is explicitly mounted as private (with MS_PRIVATE|MS_REC).

So therefore you may see references to the following command:

```
unshare -Urpf --mount-proc
```

This creates a new `mnt` namespace while mounting `/proc` for you.


### Entering A Namespace

In order to reduce complexity, I have exited the namespaces created earlier. I have created a new namespace with the following command
```
unshare -Urfp --mount-proc
```

I have also created a *different* sleep process just to help identify the namespace. Since I only have a single new namespace I can use the `lsns` command to identify the correct pid:

```
[ user@localhost  ~]$ lsns |grep bash
4026532965 pid         2 13142 user -bash
```


Then run the `nsenter` command:

```
sudo nsenter -t 13142 -a
```

The `-a` flag tells the `nsenter` command to enter all namespaces of that PID. `sudo` is required with the `-a` flag or else you will not be able to change to all the appropriate namespaces. You should now be able to list all the PIDS in this NS:

```
[ root@localhost  ~]$ ps -ef

UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 09:54 pts/0    00:00:00 -bash
root           8       1  0 09:54 pts/0    00:00:00 sleep 99999
root          25       0  0 10:15 pts/1    00:00:00 -bash
root          31      25  0 10:15 pts/1    00:00:00 ps -ef
```

## Wrapping Up

The `PID` namespace is an important one when it comes to building isolated environments. It allows processes to have their own PIDs regardless of the host system. In a world where multiple hosts may be involved in orchestrating isolated environments (containers), it becomes crucial to have a facility that guarantees unique PIDs when freezing and migrating processes. On top of that, for security reasons if you are running namespaces for application isolation, the `PID` namespace is vital for preventing information leaks by way of which processes a host may be running.

The `PID` namespace when combined with the `user` and `mnt` namespaces provide a great deal of protection without requiring root privileges. Modern browsers such as Firefox and Vivaldi make use of namespaces to provide browser 'sandboxing'. In the next article we'll investigate the `net` namespace and see how we can continue to construct our container by hand by adding in discrete network components.


