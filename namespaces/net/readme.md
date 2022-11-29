# Intro

Throughout the Linux Namespaces series we have taken a look at some really useful functions that are built right into the Linux Kernel. With the [UTS Namespace](https://www.redhat.com/sysadmin/uts-namespace) you can change the hostname or the NIS domain that a process reports. Largely, this is useful in the container context and not extremely useful on its own. 
	
The [PID Namespace](https://www.redhat.com/sysadmin/pid-namespace) is quite useful for isolating processes from each other. This is useful if you have a complicated environment where you are using the Linux Kernel's Freeze functionality which allows you to pause a process and resume it on a different host. It also allows for running multiple versions of an application that relies on the ability to have isolated process trees.
  
The [Mount Namespace](https://www.redhat.com/sysadmin/mount-namespaces) has a multitude of uses allowing the construction of namespaces that do not have full access to the hosts' file system. Not only is this useful for creating and managing processes on a host, it is extremely useful when creating Linux containers.
  
The [User Namespace](https://www.redhat.com/sysadmin/building-container-namespaces) allows processes to run as specific users. Among other things, this is useful for creating a namespace as an unprivileged user, but still being able to be 'root' in  the namespace. There are obvious limitations regarding the tasks these namespaces can accommodate as they do not have 'root' on the host. However, the User Namespace is most often used in combination  with the other namespaces in order to provide a greater level of isolation than would otherwise be possible.
  
In this article we're going to look at the namespace I get asked about the most: the Net Namespace. As the name implies, this namespace can be used to manage the network stack inside of a namespace. We're going to look at a few use cases for why you might want to do something like this. If you need to have two processes that use the network to communicate but you don't want the outside (or potentially even the host) to access said processes, we'll take a look at how you can use OpenVSwitch and a couple of network namespaces to accomplish this. 

We'll also take a look at how to run a DHCP server inside a different network namespace and finally how we might use a network namespace to establish a VPN connection so that you can better control which applications use the VPN and which do not; all without using an external network device!

Let's get started.

## Initial Setup

For this article I am using a CentOS Stream 9 VM. There are a few packages that need to be installed. Normally, I recommend installing everything from your package manager. In this case however, the NFV repo (at the time of writing) is populated but is not a part of the default repositories. If you try to install the meta package that normally provides these repos you will get an error:

```
[root@centos9-test ~]# dnf install centos-release-nfv-openvswitch
Updating Subscription Management repositories.
Unable to read consumer identity

This system is not registered with an entitlement server. You can use subscription-manager to register.

Last metadata expiration check: 0:45:55 ago on Wed 05 Jan 2022 09:34:06 AM CST.
No match for argument: centos-release-nfv-openvswitch
Error: Unable to find a match: centos-release-nfv-openvswitch
```

So instead I'm just going to grab the two packages we need directly from the repository. While I am at it, I will also install the DHCP packages for later on:

```
wget http://mirror.stream.centos.org/SIGs/9-stream/nfv/x86_64/openvswitch-2/Packages/o/openvswitch-selinux-extra-policy-1.0-30.el9s.noarch.rpm
wget http://mirror.stream.centos.org/SIGs/9-stream/nfv/x86_64/openvswitch-2/Packages/o/openvswitch2.15-2.15.0-51.el9s.x86_64.rpm
dnf install -y dnsmasq dhcp-client net-tools
dnf install openvswitch-selinux-extra-policy-1.0-30.el9s.noarch.rpm openvswitch2.15-2.15.0-51.el9s.x86_64.rpm
```

## Point-to-Point Net Namespaces

The first example we are going to walk through is creating two Net namespaces and have them communicate with each other. You can think of this similar to using a cross-over cable when connecting two nics together (and yes I am well aware that modern nics do not need crossover cables as they are auto-sensing these days).

In the diagram below you can see that each end of the 'cable' is labeled. I recommend using obvious names, although the naming convention you choose is completely arbitrary.

![net_namespaces-ptp.png](net_namespaces-ptp.png)

In order to accomplish this let's set up some variables to make our life easier, and the code more repeatable:

```
namespace1=client
namespace2=server
command='python3 -m http.server'
ip_address1="10.10.10.10/24"
ip_address2='10.10.10.20/24'
interface1=veth-client
interface2=veth-server
```

For this example, our server is just going to be running a basic Python3 web server. We just need to verify that the connection is working.

The first thing we need to do is to create the namespaces. Unlike other commands, the Net namespace can be accessed directly from the `ip` command which most people are already familiar with. Create the namespaces as follows:

```
ip netns add $namespace1
ip netns add $namespace2
```

You can verify that the namespaces have been created via the following command:

```
[root@centos9-test ~]# ip netns list
server
client
```

Great, we have some namespaces. This in and of itself is not very useful yet. If you run a command in one of the new namespaces you will see that there is only the loopback address and it is marked as DOWN:

```
[root@centos9-test ~]# ip netns exec $namespace2 ip ad
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Our next step is to create the virtual 'ethernet cable' by creating a link between the two namespaces. You can create a new link like so:

```
ip link add ptp-$interface1 type veth peer name ptp-$interface2
```

If you run the `ip link` command on the host you will see that there are currently 2 additional links created from this command:

```
[root@centos9-test ~]# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:57:53:77 brd ff:ff:ff:ff:ff:ff
3: ptp-veth-server@ptp-veth-client: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether d6:d9:74:95:64:6f brd ff:ff:ff:ff:ff:ff
4: ptp-veth-client@ptp-veth-server: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether f6:4a:a4:70:86:67 brd ff:ff:ff:ff:ff:ff
```

The output may look rather odd. Even though we only typed one command, since we specified a `peer` in the link command, it created a link with the following convention `<link name>@<peer link name>`. This is why I recommend picking descriptive names for your links. Once you have a few links, it can be difficult to keep track of their purpose.

At this point, the links are created but have not been assigned anywhere. Let's go ahead and assign the interfaces to their respective namespaces:

```
ip link set ptp-$interface1 netns $namespace1
ip link set ptp-$interface2 netns $namespace2
```

After running this command, the host no longer has access to these links as they have been assigned to a different Net namespace. If you rerun the `ip netns exec` command you can now see that our new namespaces have devices, but they are still marked as down and do not have IPAddresses to communicate with:

```
[root@centos9-test ~]# ip netns exec $namespace2 ip ad
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: ptp-veth-server@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether d6:d9:74:95:64:6f brd ff:ff:ff:ff:ff:ff link-netns client
```

Let's assign the IPs and bring the interfaces up:

```
ip netns exec $namespace1 ip addr add $ip_address1 dev ptp-$interface1
ip netns exec $namespace2 ip addr add $ip_address2 dev ptp-$interface2
ip netns exec $namespace1 ip link set dev ptp-$interface1 up
ip netns exec $namespace2 ip link set dev ptp-$interface2 up
```

Finally, let's start the Python3 web server in `$namespace2` and test it out:

```
ip netns exec $namespace2 $command &
```

Because we have the subnet CIDR in the `$ip_addres2` you can either do some bash trickery to remove it similar to the following:

```
ip netns exec $namespace1 curl `echo $ip_address2 |awk -F '/' '{print $1}'`:8000
```

Or simply type the the IP into the command:
```
ip netns exec $namespace1 curl 10.10.10.20:8000
```

You should receive a directory listing for whichever directory was active during the instantiation of the Python webserver. Because the original host does not have an ethernet device on the 10.x.x.x network, it cannot reach the new namespaces. Even if it did have a properly configured interface for that subnet, we used a point-to-point 'cable' meaning only the namespaces connected to that interface peering will be able to communicate with each other.

### Adding DHCP Services

Building on the examples above, we are going to create a few more variables to assist with our DHCP example:

```
ip_range_start='10.10.10.100'
ip_range_end='10.10.10.150'
netmask='255.255.255.0'
```

For DHCP to function properly we need to bring up the loopback device in the server namespace:

```
ip netns exec $namespace2 ip addr add 127.0.0.1/8 dev lo
ip netns exec $namespace2 ip link set lo up
```

We'll also want to start the dnsmasq process in the new namespace:

```
ip netns exec $namespace2 dnsmasq --interface=ptp-$interface2 --dhcp-range=$ip_range_start,$ip_range_end,$netmask
```

Finally we are going to delete the IPAddress attached to `$namespace1` and then start `dhclient` to ensure that a connection with the dhcp server can be established:

```
ip netns exec $namespace1 ip addr del $ip_address1 dev ptp-$interface1
ip netns exec $namespace1 dhclient ptp-$interface1
```

You should now see a different IP on the interface inside `$namespace1`:

```
[root@centos9-test ~]# ip netns exec $namespace1 ip ad

1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: ptp-veth-client@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 9a:61:93:29:73:d5 brd ff:ff:ff:ff:ff:ff link-netns server
    inet 10.10.10.124/24 brd 10.10.10.255 scope global dynamic ptp-veth-client
       valid_lft 3468sec preferred_lft 3468sec
    inet6 fe80::9861:93ff:fe29:73d5/64 scope link 
       valid_lft forever preferred_lft forever
```

But wait! You may have noticed that the host is behaving strangely when making DNS queries. That is because, we have done nothing to stop the network namespace from overwriting `/etc/resolv.conf`. There are a few ways you can potentially solve this. You could create a new mount namespace using the skills you learned from previous articles. In fact, this is probably the most "container-like" method of achieving the correct results. However, let me show you another way, which may be more useful if you plan to use Net namespaces on your desktop or laptop.

The `ip-netns(8)` manpage says the following:

```
For applications that are aware of network namespaces, the convention is to look for global network configuration files first in /etc/netns/NAME/ then in /etc/.  For example, if you want a different version of /etc/resolv.conf for a network namespace used to isolate your vpn you would name it /etc/netns/myvpn/resolv.conf.
```

Since we know that DHCP will overwrite `resolv.conf`, all we will have to do is touch the file in the appropriate directory. For completeness I am going to create a directory for both namespaces, but this is not necessary. The `server` namespace is not going to be receiving DHCP and thus does not need to adjust the `resolv.conf` file. Create the directories on the host file system:

```
mkdir -p /etc/netns/{$namespace1,$namespace2}
```

You can now touch `resolv.conf` for the `client` namespace:

```
touch /etc/netns/$namespace1/resolv.conf
```

With this file in place, the original Net namespace will use `/etc/resolv.conf` while the rest of the Net namespaces will reference `/etc/netns/<namespace>/resolv.conf`.


## Multiple Net Namespaces With Open vSwitch

While it is good to be able to have services running locally that only talk to each other, most containerized workloads involve multiple services working together to achieve a goal. Just like with physical computers, in order to eloquently connect multiple namespaces to the same network, we need to deploy a switch. There are multiple virtual switches that can be used for this purpose. For this demonstration I'm going to be deploying Open vSwitch. Before proceeding, I recommend either removing all the namespaces you have previously created, or simply reboot your host (as namespaces do not survive a reboot by default). We could use the DHCP tactic we discussed above, but it is slightly easier to set static IPs again. Below are the variable declarations:

```
namespace1=east
namespace2=west
namespace3=central
command='python3 -m http.server'
ip_address1="10.10.10.10/24"
ip_address2='10.10.10.20/24'
ip_address3='10.10.10.30/24'
interface1=east
interface2=west
interface3=central
```

Let's create the namespaces as we did before:

```
ip netns add $namespace1
ip netns add $namespace2
ip netns add $namespace3
```

We are going to start Open vSwitch and then create a bridge for networking:

```
systemctl start openvswitch
ovs-vsctl add-br NAMESPACE-DEMO
```

Just as before we are going to create some `veth peer` interfaces, but this time we are only going to specify one side of the "cable". That is because we will be adding the other end of the 'cable' to the virtual switch. When we are done it will look something like this:

![net_namespaces-ovs.png](net_namespaces-ovs.png)

Adding the interfaces should look very familiar by now:

```
ip link add $interface1 type veth peer name ovs-$interface1
ip link set $interface1 netns $namespace1
ip link add $interface2 type veth peer name ovs-$interface2
ip link set $interface2 netns $namespace2
ip link add $interface3 type veth peer name ovs-$interface3
ip link set $interface3 netns $namespace3
```

You can verify that the commands worked as expected by taking a look inside of one of the namespaces:

```
[root@centos9-test ~]# ip netns exec $namespace1 ip link

1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6: east@if5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 32:5f:80:91:34:40 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

The next step is to add the ports onto the switch itself:

```
ovs-vsctl add-port NAMESPACE-DEMO ovs-$interface1
ovs-vsctl add-port NAMESPACE-DEMO ovs-$interface2
ovs-vsctl add-port NAMESPACE-DEMO ovs-$interface3
```

You can also view the ports that are being used on your Open vSwitch:

```
[root@centos9-test ~]# ovs-vsctl list-ports NAMESPACE-DEMO
ovs-central
ovs-east
ovs-west
```

As you have seen from the previous outputs, the interfaces exist and are attached to their respective components. However, they have neither an IP Address nor are they currently UP. You can accomplish this similar to before with the following commands:

```
ip netns exec $namespace1 ip addr add $ip_address1 dev $interface1
ip netns exec $namespace2 ip addr add $ip_address2 dev $interface2
ip netns exec $namespace3 ip addr add $ip_address3 dev $interface3

ip netns exec $namespace1 ip link set dev $interface1 up
ip netns exec $namespace2 ip link set dev $interface2 up
ip netns exec $namespace3 ip link set dev $interface3 up

ip link set dev ovs-$interface1 up
ip link set dev ovs-$interface2 up
ip link set dev ovs-$interface3 up
```

Now let's do some quick `ping` tests to verify that the namespaces are functioning as expected:

```
ip netns exec $namespace3 ping -c 2 10.10.10.20
ip netns exec $namespace1 ping -c 2 10.10.10.20
```

Finally, view the arp table of each namespace:

```
[root@centos9-test ~]# ip netns exec $namespace1 arp
Address                  HWtype  HWaddress           Flags Mask            Iface
10.10.10.20              ether   ce:29:53:b3:b1:bd   C                     east

[root@centos9-test ~]# ip netns exec $namespace2 arp
Address                  HWtype  HWaddress           Flags Mask            Iface
10.10.10.10              ether   32:5f:80:91:34:40   C                     west
10.10.10.30              ether   72:4c:7b:f3:87:fe   C                     west

[root@centos9-test ~]# ip netns exec $namespace3 arp

Address                  HWtype  HWaddress           Flags Mask            Iface
10.10.10.20              ether   ce:29:53:b3:b1:bd   C                     central
```

You can see that since each namespace pinged the `west` namespace, it has entries in its arp table for both the other IPs whereas both east and central only have a single entry in their arp table. 

Great! So we have a bunch of namespaces that can communicate with each other via Open vSwitch. If you want to enable host-to-namespace communication as well you can follow a similar pattern: 
1. Create the `veth peer`
2. Add an IP to the host end of the 'cable'
3. Set the host veth interface up
4. Connect the other end of the 'cable' to Open vSwitch
5. Set the Open vSwitch port up

The follow commands will do that:

```
ip link add host-if type veth peer name ovs-host-if
ip addr add 10.10.10.40/24 dev host-if
ip link set dev host-if up
ovs-vsctl add-port NAMESPACE-DEMO ovs-host-if
ip link set dev host-if up
```

You should now be able to communicate with the processes inside of all the namespaces connected to Open vSwitch.

```
ping -c 2 10.10.10.20
```

## Using A Net Namespace For VPNs

Now that we have explored connecting all of the namespaces, let's explore creating a single namespace with a OpenVPN connection. There is a good example up on [James MCM's github](https://gist.github.com/jamesmcm/f8d6e9f290f7b128e1b500430789d651) that we will be deriving the work from. For our purposes, I am going to assume you are not running a firewall on your local machine. Like before, we are going to set up some variables to make this more easily adaptable to your situation. However, I am going to take this opportunity to demonstrate a few different facets of the Net Namespace. Ultimately, performing this task is much more suited to a language other than Bash, which you will discover below. In order to accomplish our goal, we will need to have 2 different sessions attached to the host where we will be doing our work. I will make denotations when work is being done on the host (i.e. in the root namespace) and when it is being done in our new namespace.

### Lay The Groundwork

We will be starting off in a shell in the root namespace. Unlike other objectives in this series, we will be having to use elevated privileges as I know of no real way to modify network interfaces in the root namespace without using `sudo` or full root access.

Start by setting some environment variables in order to make this more easily adjustable for your needs:
```
#on the host 
export VPN_NAMESPACE=vpn-ns
VETH1=host
VETH2=vpn-namespace
VETH1_IP=172.25.0.1
VETH2_IP=172.25.0.2
SUBNET_BITS=24
VPN_NETWORK=172.25.0.0/$SUBNET_BITS
MAIN_INTERFACE=`ip route |grep default |awk '{print $5}'`
```

As mentioned above, you can create a directory under `/etc/netns` for your namespace in order to help with `resolv.conf` issues. In this case, we are using this directory to store a copy of the `resolv.conf` file that we will mount later on.


```
sudo mkdir -p /etc/netns/$VPN_NAMESPACE
sudo chown -R $USER /etc/netns/$VPN_NAMESPACE/
echo "nameserver 37.235.1.174" > /etc/netns/$VPN_NAMESPACE/resolv.conf
```

> The name server 37.235.1.174 is listed on [freedns.zone](freedns.zone) which claims to not do any logging. While no one can verify this short of a court order, I prefer these servers to Google's 8.8.8.8 which is well known for harvesting data.
{.is-info}


The keen observer might have noticed that we are prepopulating the `resolv.conf` file. Normally, this is handled by NetworkManager or `resolvd`. However, in this case, we will be using OpenVPN on the command line, and while it does have `up` and `down` sections in its' `.ovpn` files, for simplicity, our script is only going to be mounting the prepopulated `resolv.conf`. The below command will create a script that will be run when OpenVPN makes a connection to a specific endpoint. This is defined per VPN connection and therefore will not be called unless added to each `.ovpn` file.



```
sudo -Eu root bash -c 'cat << EOF > /etc/openvpn/set_container_resolver.sh
#!/usr/bin/env bash
case \$script_type in
up)
    mount -o bind /etc/netns/${VPN_NAMESPACE}/resolv.conf /etc/resolv.conf
    ;;
down)
    umount /etc/resolv.conf
    ;;
esac
exit 0
EOF'

sudo chmod +x /etc/openvpn/set_container_resolver.sh
```

Next, open a new terminal. This new terminal will create the new namespace from which the OpenVPN connection will be established. The environment variables are almost the same as above, except the `VPN_NETWORK` and `MAIN_INTERFACE` are replaced by the `PS1` variable. The `PS1` variable controls the prompt of the bash shell. This is cosmetic and is meant to help distinguish the different namespaces.


```
# open a new terminal/shell
unshare --user --map-root-user --net --mount --fork 

VPN_NAMESPACE=vpn-ns
VETH1=host
VETH2=vpn-namespace
VETH1_IP=172.25.0.1
VETH2_IP=172.25.0.2
SUBNET_BITS=24
PS1='\u@${VPN_NAMESPACE} $'
```

We are going to create the network devices. The first line actually creates the VETH pair from within the new namespace and connects it to the namespace associated with `pid 1`.

> The `ip link` command has a subcommand called `netns`. We have used it a few times already to create a namespace and give it a name. `netns` can also take a pid. This is useful when the namespace has no name.
{.is-info}


```
ip link add $VETH2 netns $$ type veth peer name $VETH1 netns 1
ip addr add 127.0.0.1/8 dev lo
ip link set lo up

ip link set $VETH2 up
ip addr add $VETH2_IP/$SUBNET_BITS dev $VETH2
ip route add default via $VETH1_IP dev $VETH2
```

While we can attach a VETH device to the root namespace, we cannot set it `up` nor can we give it an IP. This requires elevated privileges in the root namespace. Additionally, it is likely that your system does not have IP Forwarding enabled. The following commands, issued in the root namespace, will accomplish both of these tasks:


```
# on the host
sudo ip link set $VETH1 up
sudo ip addr add $VETH1_IP/$SUBNET_BITS dev $VETH1

sudo iptables -t nat -A POSTROUTING -s $VPN_NETWORK -o $MAIN_INTERFACE -j MASQUERADE
sudo sysctl -q net.ipv4.ip_forward=1
```

This next part is optional. As we have created a `net` namespace with the `unshare` command, it will not be available or visible to the `ip netns` command. This is because `unshare` creates its processes in `/proc/<pid>` while the `ip netns` command expects namespaces to be located in `/run/netns/<namespace>`. There are a few ways to get around this problem, but perhaps the simplest is to *bind mount* `/proc/` into `/run/netns` effectively having the namespace mounted in both locations. 

The reason you may wish to do this is that creating a namespace with just the `ip netns` command only creates the network namespace. It also does so as root. This means that if you are looking for an extra layer of isolation via the `user` namespace, you cannot use the `ip netns` command. However, the `ip` command provides a very convenient way to interact with already running namespaces. The below commands will *bind mount* the **vpn-ns** namespace we are using:

```
sudo mkdir -p /run/netns
sudo -Eu root bash -c 'touch /run/netns/$VPN_NAMESPACE'
sudo -Eu root bash -c "mount -o bind /proc/`lsns |grep unshare |awk '{print $4}' |uniq`/ns/net /run/netns/$VPN_NAMESPACE"
```

### Test The VPN

At this point run the following command in both the `net` namespace and the root namespace:
```
curl ifconfig.co/city
```

This should return the same result (whether totally accurate or not). We will use this result to verify if the VPN connection is successfully isolated to a specific namespace. 

What follows below is a brief overview of an OpenVPN file. As this is not intented to be an OpenVPN tutorial, we will not dive in very deep. In theory the same steps should be able to be used for Wireguard, PPTP or most other VPN protocols supported by Linux.

#### OpenVPN Config Files

OpenVPN uses files commonly post-fixed with `.ovpn`. As with most files in Linux, the file extension is meant for human readibility and not for the program itself. You could very easily have a `.txt` file or no extension at all and OpenVPN would not care as long as the contents were what it was expecting. Here is the OpenVPN file I am using with some details such as certificates and remote IP/URLs altered or removed:

```
script-security 2
up /etc/openvpn/set_container_resolver.sh
down /etc/openvpn/set_container_resolver.sh
dev tun
persist-tun
persist-key
cipher AES-256-CBC
auth SHA256
tls-client
client
resolv-retry infinite
remote some.example.com 1194 udp
verify-x509-name "vpn-ns-demo" name
auth-user-pass
remote-cert-tls server
comp-lzo adaptive

<ca>
-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----
</ca>

<cert>
-----BEGIN CERTIFICATE-----
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
-----END PRIVATE KEY-----
</key>

key-direction 1

<tls-auth>

-----BEGIN OpenVPN Static key V1-----
-----END OpenVPN Static key V1-----
</tls-auth>
```

The important bits for this tutorial are at the top. The `up` and `down` sections of the file reference the script we created earlier which mounts and unmounts the `resolv.conf` file. Without this, the `resolv.conf` does not get updated and, depending on the VPN endpoint, means that internet traffic will not resolve. The [OpenVPN website](https://openvpn.net/community-resources/how-to/#creating-configuration-files-for-server-and-clients) has some pointers for getting started with the client configuration files if you are interested in reading further.

#### Testing the VPN

Inside the namespace start the VPN:
```
openvpn --config Downloads/pfSense-UDP4-1194-stratusvpn-config.ovpn
```

> As I am not feeding a credentials file into the `openvpn` command, it will consume the foreground shell after you log in. This is part of the reason it is useful to *bind mount* the namespace where the `ip netns` command can interact with it.
{.is-info}

There are two ways you can interact with the namespace now. You can find the `pid` and use `nsenter`:

```
sudo nsenter  -t `lsns |grep unshare |awk '{print $4}' |uniq`
```

This will give you a root shell where you can run your curl command:

```
[root@stratus-desktop /]# curl ifconfig.co/city
Windsor
```

> NOTE: The shell prompt did not change because the `PS1` variable is tied to the bash session as it was not globally exported
{.is-danger}

Confirm that the root namespace is registering in a different city:

```
stratus@stratus-desktop ~  $ curl ifconfig.co/city
Sioux Falls
```

This will work fine for several programs, but not for a browser. Trying to launch a browser results in odd behaviour:

```
[root@stratus-desktop /]# firefox
Running Firefox as root in a regular user's session is not supported.  ($HOME is /root which is owned by nobody.)
```

This is the real reason we *bind mounted* the namespace. We can get around this by using the `ip netns` command:

```
# On the host
sudo ip netns exec $VPN_NAMESPACE sudo -u $USER firefox
```

You now have a browser running in a namespace that is sending all of its' traffic out the VPN connection while having the rest of the system continue to use the local connection.

# Wrapping Up

This was a big topic. We covered a lot in this post:

1. Having two isolated `net` namespaces communicate directly with each other
2. Standing up DHCP instead of static IPs
3. We created multiple `net` namespaces and used OpenvSwitch to connect them all together
4. Created a `net` namespace we could use for VPN communication so that only applications run within the namespace are routed through the VPN

We learned several techniques for creating and interacting with the `net` namespace and we combine this with previous knowledge of the `user` and `mnt` namespaces to further isolate processes. You can see how these namespaces: `pid`, `user`, `mnt` and `net`, can combine to make a very powerful isolation layer on any Linux system. True, Linux containers provide this functionality in a more user friendly way, but it is always important to grasp the fundamental building blocks. 

Whereas containers are supposed to only have a single process by design (although it is certainly possible to have more), creating your own namespaces allows you the flexibility to customize your environment to your specific needs all without downloading or running binaries you didn't intentionally install. For tasks such as isolating application traffic, namespaces provide a very customizable interface to achieving these goals.

