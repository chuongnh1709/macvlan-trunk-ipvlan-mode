
# Macvlan, Ipvlan and 802.1q Trunk Driver Notes

### Contents

 - [MacVlan Bridge Mode Example Usage](#macvlan-bridge-mode-example-usage)

 - [Ipvlan L2 Mode Example Usage](#ipvlan-l2-mode-example-usage)

 - [Macvlan 802.1q Trunk Bridge Mode Example Usage](#macvlan-8021q-trunk-bridge-mode-example-usage)

 - [Ipvlan 802.1q Trunk L2 Mode Example Usage](#ipvlan-8021q-trunk-l2-mode-example-usage)

 - [IPVlan L3 Mode Example](#ipvlan-l3-mode-example)

 - [IPv6 Macvlan Bridge Mode](#ipv6-macvlan-bridge-mode)

 - [IPv6 Ipvlan L3 Mode](#ipv6-ipvlan-l2-mode)

 - [IPv6 Ipvlan L3 Mode](#ipv6-ipvlan-l3-mode)

 - [Manually Creating 802.1q Links](#manually-creating-8021q-links)


Branch at: [PR#964](https://github.com/docker/libnetwork/pull/964).

- See this script: [ipvlan-macvlan-it.sh](https://github.com/nerdalert/dotfiles/blob/master/ipvlan-macvlan-it.sh) for a list of tests with 50+ different macvlan/ipvlan networking scenario that you can copy and paste to give the drivers a whirl. 
- As the options change around a bit in experimental there may be a need for deleting the local driver boltdb k/v store. To do this simply stop the docker daemon, delete the network files `rm  /var/lib/docker/network/files/*` and start the docker daemon back up.
- The driver caches `NetworkCreate` callbacks to the boltdb datastore along with populating `*networks`. In the case of a restart, the driver initializes the datastore with `Init()` and populates `*networks` since `NetworkCreate()` is only called once. 
- There can only be one (ipvlan or macvlan) driver type bound to a host interface with running containers at any given time. Currently the driver does not prevent ipvlan and macvlan networks to be created with the same `-o parent` but will throw an error if you try to start an ipvlan container and a macvlan container at the same time on the same `-o parent`. A mix of host interfaces and macvlan/ipvlan types can be used with running containers, each interface just needs to use the same type. Example: Macvlan Bridge or IPVlan L2. There is no mixing of running containers on the same host interface. There are also implications mixing Ipvlan L2 & L3 simultaneously as L3 takes a NIC out of promiscous mode. For more information you can tail `dmesg` logs as you create networks & run containers. 
- The specified gateway is external to the host or at least not defined by the driver itself. 
- Each network is isolated from one another. Any container inside the network/subnet can talk to one another without a reachable gateway in both `macvlan bridge` mode and `ipvlan L2` mode. IP tables may be able to work around that if a user wanted to.
- Containers on separate networks cannot reach one another without an external process routing between the two networks/subnets.
- **Note:** In both Macvlan and Ipvlan you are not able to ping or communicate with the default namespace IP address. For example, if you create a container and try to ping the Docker host's `eth0` it will **not** work. That traffic is explicitly filtered by the kernel modules themselves to offer additional provider isolation and security.

- More information about Ipvlan & Macvlan can be found in the upstream [readme](https://github.com/torvalds/linux/blob/master/Documentation/networking/ipvlan.txt)

- If you just want to test the Docker binary with the drivers compiled in download: [docker-1.11.0-dev.zip](https://github.com/nerdalert/cloud-bandwidth/files/160301/docker-1.11.0-dev.zip). The driver's persistent database ([boltdb](https://github.com/boltdb/bolt)) data models are subject to change while the drivers are still under review/development. As they change if you created a network with an older model you may see some nil value errors when you start docker and an old model from an existing network created by the macvlan or ipvlan drivers get populated. You can reset the k/v boltdb database by simply deleting the datastore file with `rm /var/lib/docker/network/files/local-kv.db`.

- The Docker build is the following:

```
docker -v
Docker version 1.11.0-dev, build ac9d1b7-unsupported
```

1. Download the zipped experimental binary above or here: [docker-1.11.0-dev.zip](https://github.com/nerdalert/dotfiles/files/160936/docker-1.11.0-dev.zip)
2. Unzip it on a Linux OS.
3. Stop any other docker instances `killall docker`
4. `./docker-1.11.0-dev daemon` (optionally, add `-D` for extra debugging logs).
5. Run the examples below or for the most up to date ones see this script: [ipvlan-macvlan-it.sh](https://github.com/nerdalert/dotfiles/blob/master/ipvlan-macvlan-it.sh)

### MacVlan Bridge Mode Example Usage ###

- In this example, `eth0` on the docker host has an IP on the `192.168.1.0/24` network and a default gateway of `192.168.1.1`. The gateway is an external router with an address of `192.168.1.1`. An IP address is not required on the Docker host interface `eth0` in `bridge` mode, it merely needs to be on the proper upstream network to get forwarded by a network switch or network router.

<a href="url"><img src="https://cloud.githubusercontent.com/assets/1711674/13232710/5c1c7d2a-d97e-11e5-99e9-f19d1aa0f87f.jpg" align="center" width="350" ></a>

**Note** For Macvlan bridge mode and Ipvlan L2 mode the subnet values need to match the NIC's interface of the Docker host. For example, Use the same subnet and gateway of the Docker host ethernet interface that is specified by the `-o parent=` option.

- The parent interface used in this example is `eth0` and it is on the subnet `192.168.1.0/24`. The containers in the `docker network` will also need to be on this same subnet as the parent `-o parent=`. The gateway is an external router on the network, not any ip masquerading or any other local proxy (see [diagrams for topologies](https://cloud.githubusercontent.com/assets/1711674/13032315/099ddbb2-d2bb-11e5-86c1-a526ade612cc.jpg)).

- The driver is specified with `-d driver_name` option. In this case `-d macvlan`

- The parent interface `-o parent=eth0` is configured as followed:

```
ip addr show eth0
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.1.251/24 brd 192.168.1.255 scope global eth0
```

Create the macvlan network and run a container attaching to it:

```
# Macvlan  (--macvlan_mode= Defaults to Bridge mode if not specified)
docker network  create  -d macvlan  --subnet=192.168.1.0/24 --gateway=192.168.1.1 -o parent=eth0 macnet100
docker  run --net=macnet100 -it --rm alpine /bin/sh
```

You can explicitly specify the `bridge` mode option `-o macvlan_mode=bridge`. It is the default so will be in `bridge` mode either way.



### Ipvlan L2 Mode Example Usage ###

The ipvlan `L2` mode example is virtually identical to the macvlan `bridge` mode example. The driver is specified with `-d driver_name` option. In this case `-d ipvlan`

<a href="url"><img src="https://cloud.githubusercontent.com/assets/1711674/13232749/7999fdf0-d97e-11e5-9e62-8793ccdcb061.jpg" align="center" width="350" ></a>


The parent interface `-o parent=eth0` is configured as followed:

```
ip addr show eth0
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.1.251/24 brd 192.168.1.255 scope global eth0
```

Create the ipvlan network and run a container attaching to it:

```
# Ipvlan  (-o ipvlan_mode= Defaults to L2 mode if not specified)
docker network  create -d ipvlan  --subnet=192.168.1.0/24 --gateway=192.168.1.1 -o parent=eth0 ipnet100
docker  run --net=ipnet100 -it --rm alpine /bin/sh
```

You can explicitly specify the `l2` mode option `-o ipvlan_mode=l2`. It is the default so will be in `l2` mode either way.




### Macvlan 802.1q Trunk Bridge Mode Example Usage ###

<a href="url"><img src="https://cloud.githubusercontent.com/assets/1711674/13232776/9e93b966-d97e-11e5-8803-e30e0d913e47.jpg" align="center" width="350" ></a>


Start two containers to test a ping. The default namespace is not reachable per macvlan design in order to isolate container namespaces from the underlying host.

The Linux subinterface tagged with a vlan can either already exist or will be created when you call a `docker network create`. `docker network rm` will delete the subinterface. Parent interfaces such as `eth0` are not deleted, only subinterfaces with a netlink parent index > 0.

For the driver to add/delete the vlan subinterfaces the format needs to be `interface_name.vlan_tag`. 

For example: `eth0.50` denotes a parent interface of `eth0` with a slave of `eth0.50` tagged with vlan id `50`. The equivalent `ip link` command would be `ip link add link eth0 name eth0.50 type vlan id 50`.

Replace the `macvlan` with `ipvlan` in the `-d` driver argument to create macvlan 802.1q trunks. 

**Vlan ID 50**

In the first network tagged and isolated by the Docker host, `eth0.50` is the parent interface tagged with vlan id `50` specified with `-o parent=eth0.50`. Other naming formats can be used, but the links need to be added and deleted [manually](https://gist.github.com/nerdalert/28168b016112b7c13040#manually-creating-8021q-links) using `ip link` or Linux configuration files. As long as the `-o parent` exists anything can be used if compliant with Linux netlink.

```
# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged
docker network  create  -d macvlan  --subnet=192.168.50.0/24 --gateway=192.168.50.1 -o parent=eth0.50 macvlan50

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=macvlan50 -it --name macvlan_test5 --rm alpine /bin/sh
docker run --net=macvlan50 -it --name macvlan_test6 --rm alpine /bin/sh
```

**Vlan ID 60**

In the second network, tagged and isolated by the Docker host, `eth0.60` is the parent interface tagged with vlan id `60` specified with `-o parent=eth0.60`. The `macvlan_mode=` defaults to `macvlan_mode=bridge`. It can also be explicitly set with the same result as shown in the next example.

```
# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged. 
docker network  create  -d macvlan  --subnet=192.168.60.0/24 --gateway=192.168.60.1 -o parent=eth0.60 -o --macvlan_mode=bridge macvlan60

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=macvlan60 -it --name macvlan_test7 --rm alpine /bin/sh
docker run --net=macvlan60 -it --name macvlan_test8 --rm alpine /bin/sh
```

**Example:** Multi-Subnet Macvlan 802.1q Trunking

The same as the example before except there is an additional subnet bound to the network that the user can choose to provision containers on. In MacVlan/Bridge mode, containers can only ping one another if they are on the same subnet/broadcast domain unless there is an external router that routes the traffic (answers ARP etc) between the two subnets.

```
### Create multiple bridge subnets with a gateway of x.x.x.1:
docker network  create  -d macvlan \
	--subnet=192.168.164.0/24 --subnet=192.168.166.0/24 \
	--gateway=192.168.164.1  --gateway=192.168.166.1 \
	 -o parent=eth0.166 \
	 -o macvlan_mode=bridge macvlan64

docker run --net=macvlan64 --name=macnet54_test --ip=192.168.164.10 -itd alpine /bin/sh
docker run --net=macvlan64 --name=macnet55_test --ip=192.168.166.10 -itd alpine /bin/sh
docker run --net=macvlan64 --ip=192.168.164.11 -it --rm alpine /bin/sh
docker run --net=macvlan64 --ip=192.168.166.11 -it --rm alpine /bin/sh
```

### Ipvlan 802.1q Trunk L2 Mode Example Usage ###

Architecturally, Ipvlan L2 mode trunking is the same as Macvlan with regard to gateways and L2 path isolation. There are nuances that can be advantageous for CAM table exhaustion in ToR switches, one MAC per port and MAC exhaustion on a host's parent NIC to name a few.

<a href="url"><img src="https://cloud.githubusercontent.com/assets/1711674/13232776/9e93b966-d97e-11e5-8803-e30e0d913e47.jpg" align="center" width="350" ></a>

The Linux subinterface tagged with a vlan can either already exist or will be created when you call a `docker network create`. `docker network rm` will delete the subinterface. Parent interfaces such as `eth0` are not deleted, only subinterfaces with a netlink parent index > 0.

For the driver to add/delete the vlan subinterfaces the format needs to be `interface_name.vlan_tag`. Other subinterface naming can be used as the spacified parent, but the link will not be deleted automatically when `docker network rm` is invoked. 

The option to use either existing parent vlan subinterfaces or let Docker manage them enables the user to either completely manage the Linux interfaces and networking or let Docker create and delete the Vlan parent subinterfaces (netlink `ip link`) with no effort from the user.

For example: `eth0.10` or `eth0:10` to denote a subinterface of `eth0` tagged with vlan id `10`. The equivalent `ip link` command would be `ip link add link eth0 name eth0.10 type vlan id 10`.

The example creates the vlan tagged networks and then start two containers to test connectivity between containers. Different Vlans cannot ping one another without a router routing between the two networks. The default namespace is not reachable per ipvlan design in order to isolate container namespaces from the underlying host.

**Vlan ID 20**

In the first network tagged and isolated by the Docker host, `eth0.20` is the parent interface tagged with vlan id `20` specified with `-o parent=eth0.20`. Other naming formats can be used, but the links need to be added and deleted manually using `ip link` or Linux configuration files. As long as the `-o parent` exists anything can be used if compliant with Linux netlink.

```
# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged
docker network  create  -d ipvlan  --subnet=192.168.20.0/24 --gateway=192.168.20.1 -o parent=eth0.20 ipvlan20

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=ipvlan20 -it --name ivlan_test1 --rm alpine /bin/sh
docker run --net=ipvlan20 -it --name ivlan_test2 --rm alpine /bin/sh
```

**Vlan ID 30**

In the second network, tagged and isolated by the Docker host, `eth0.30` is the parent interface tagged with vlan id `30` specified with `-o parent=eth0.30`. The `ipvlan_mode=` defaults to l2 mode `ipvlan_mode=l2`. It can also be explicitly set with the same result as shown in the next example.

```
# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged. 
docker network  create  -d ipvlan  --subnet=192.168.30.0/24 --gateway=192.168.30.1 -o parent=eth0.30 -o ipvlan_mode=l2 ipvlan30

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=ipvlan30 -it --name ivlan_test3 --rm alpine /bin/sh
docker run --net=ipvlan30 -it --name ivlan_test4 --rm alpine /bin/sh
```

The gateway is set inside of the container as the default gateway. That gateway would typically be an external router on the network.

```
$ ip route
default via 192.168.30.1 dev eth0
192.168.30.0/24 dev eth0  src 192.168.30.2
```

Example: Multi-Subnet Ipvlan L2 Mode starting two containers on the same subnet and pinging one another. In order for the `192.168.114.0/24` to reach `192.168.116.0/24` it requires an external router in L2 mode. L3 mode can route between subnets that share a common `-o parent=`. This same multi-subnet example is also valid for Macvlan `bridge` mode.

Secondary addresses on network routers are common as an address space becomes exhausted to add another secondary to a L3 vlan interface or commonly refered to as a "switched virtual interface" (SVI).

```
docker network  create  -d ipvlan \
	--subnet=192.168.114.0/24 --subnet=192.168.116.0/24 \
	--gateway=192.168.114.254  --gateway=192.168.116.254 \
	 -o parent=eth0.114 \
	 -o ipvlan_mode=l2 ipvlan114
	 
docker run --net=ipvlan114 --ip=192.168.114.10 -it --rm alpine /bin/sh
docker run --net=ipvlan114 --ip=192.168.114.11 -it --rm alpine /bin/sh
```

A key takeaway is operators have the ability to map their physical network into their virtual network for integrating containers into their environment with no opertaional overhauls required. 

NetOps simply drops an 802.1q trunk into the Docker host or a bonded multi-link aggregation pair of connections to a top of rack that get bonded using LACP for example to create one virtual link. That virtual link would be the `-o parent` passed in the network creation. For single links it is as simple as `-o parent=eth0` for untagged or `-o parent=eth0.10` for a tag of VLAN 10.

### IPVlan L3 Mode Example

IPVlan will require routes to be distributed to each endpoint. The driver only builds the Ipvlan L3 mode port and attaches the container to the interface. Route distribution throughout a cluster is beyond the scope of the initial driver. That said, here is some information for those curious how Ipvlan L3 will fit into container networking.

-Ipvlan L3 mode drops all broadcast and multicast traffic. 
-L3 mode needs to be on a separate subnet as the default namespace since it requires a netlink route in the default namespace pointing to the Ipvlan parent interface. 
- The parent interface used in this example is `eth0` and it is on the subnet `192.168.1.0/24`. Notice the `docker network` is **not** on the same subnet as `eth0`.
- Unlike macvlan bridge mode and ipvlan l2 modes, different subnets/networks can ping one another as long as they share the same parent interface `-o parent=`.

```
ip a show eth0
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:50:56:39:45:2e brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.251/24 brd 192.168.1.255 scope global eth0
```

-A traditional gateway doesn't mean much to an L3 mode Ipvlan interface since there is no broadcast traffic allowed. Because of this we simply point the the containers `eth0` device as the default gateway. See below for CLI output of `ip route` from inside an L3 container. 

The mode ` -o ipvlan_mode=l3` must be explicitly specified since the default ipvlan mode is `l2`.

```
# Create the Ipvlan L3 network
docker network  create -d ipvlan \
    --subnet=192.168.120.0/24 \
    -o parent=eth0 \
    -o ipvlan_mode=l3 ipvlan120
    
# Run a container on the new network
docker run --net=ipvlan120 -it --rm alpine /bin/sh 

# Or specify a specific IPv4 address
docker run --net=ipvlan120 --ip=192.168.120.10 -it --rm alpine /bin/sh 
```

You will notice there is no `--gateway=` option in the network create. The field is ignored if one is specified since it doesn't have a use in `l3` mode. Take a look at the container routing table from inside of the container:

```
# Inside the container
$ ip route
default dev eth0
192.168.120.0/24 dev eth0  src 192.168.120.2
```


In order to ping the container from a remote host or the container be able to ping a remote host, the remote host needs to have a route pointing to the host IP address of the container's Docker host eth interface. 

-If you have two hosts and one of them has a ipvlan l3 mode container running, the second host needs to have a route added to ping the container. The following are the addresses of the example of two hosts, host1 running docker with a container in ipvlan l3 mode and *host2* a regular host we will be pinging from the container and vice versa:

- **Host1:** Docker Host **eth0:** 192.168.1.249/24 **container network:** 192.168.130.0/24 **L3 mode container IP:** 192.168.130.2

- **Host2:** Remote Host **eth0:** 192.168.1.251/24


First, add a route to the remote host that tells the host, in order to reach the container network `192.168.130.0/24` on *Host1* use the next hop *Host1* eth0 `192.168.1.249`. By adding the following route to the remote host, **host2**

```
# On Host2 add a route for the L3 container network to be reached via the Docker host eth0 address (192.168.1.249).
ip route add 192.168.130.0/24 via 192.168.1.249
```

After adding the above route to *Host1* it should be able to ping the L3 mode container at `192.168.130.2` from its eth0 interface with the IP `192.168.1.251` as should the container be able to ping `192.168.1.249` from its address at `192.168.130.2`.

L3 (layer 3) or ip routing has the ability to scale well beyond L2 (layer 2) networks. The Internet is made up of a collection of interconnected L3 networks. This is attractive when coupled with the density presented by migrations to Docker containers and worth spending some time to understand if new to scaling networks.

**Example:** Multi-Subnet Ipvlan L3 Network

```
docker network  create  -d ipvlan \
	--subnet=192.168.110.0/24 --subnet=192.168.112.0/24 \
	--gateway=192.168.110.1  --gateway=192.168.112.1 \
	 -o parent=eth0.110 \
	 -o ipvlan_mode=l3 ipvlan110

# Container #1
docker run --net=ipvlan110 --name=ipnet110_test --ip=192.168.110.10 -itd alpine /bin/sh
# Container #2
docker run --net=ipvlan110 --name=ipnet112_test --ip=192.168.112.10 -itd alpine /bin/sh

```

Once both are started, ping from one to the other:

- Inside of Container #1

```
/ # ip a show eth0
82: eth0@if81: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 00:50:56:2b:29:40 brd ff:ff:ff:ff:ff:ff
    inet 192.168.110.10/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe2b:2940/64 scope link
       valid_lft forever preferred_lft forever
/ #
/ # ip route
default dev eth0
192.168.110.0/24 dev eth0  src 192.168.110.10
/ #
/ # ping 192.168.112.10
PING 192.168.112.10 (192.168.112.10): 56 data bytes
64 bytes from 192.168.112.10: seq=0 ttl=64 time=0.060 ms

--- 192.168.112.10 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.060/0.060/0.060 ms
```

Inside of Container #2

```
/ # ip a show eth0
83: eth0@if81: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 00:50:56:2b:29:40 brd ff:ff:ff:ff:ff:ff
    inet 192.168.112.10/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe2b:2940/64 scope link
       valid_lft forever preferred_lft forever
/ #
/ # ip route
default dev eth0
192.168.112.0/24 dev eth0  src 192.168.112.10
/ #
/ # ping 192.168.110.10
PING 192.168.110.10 (192.168.110.10): 56 data bytes
64 bytes from 192.168.110.10: seq=0 ttl=64 time=0.137 ms

--- 192.168.110.10 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.137/0.137/0.137 ms
```

### IPv6 Macvlan Bridge Mode

**Example:** Macvlan Bridge mode, 802.1q trunk, VLAN ID: 218, Multi-Subnet, Dual Stack

```
### Create multiple bridge subnets with a gateway of x.x.x.1:
docker network  create  -d macvlan \
	--subnet=192.168.216.0/24 --subnet=192.168.218.0/24 \
	--gateway=192.168.216.1  --gateway=192.168.218.1 \
	--subnet=fe94::/64 --gateway=fe94::10 \
	 -o parent=eth0.218 \
	 -o macvlan_mode=bridge macvlan216

docker run --net=macvlan216 --name=macnet216_test --ip=192.168.216.10 -itd debian
docker run --net=macvlan216 --name=macnet216_test --ip=192.168.218.10 -itd debian
docker run --net=macvlan216 --ip=192.168.216.11 -it --rm debian
docker run --net=macvlan216 --ip=192.168.218.11 -it --rm debian
```


View the details of one of the containers

```
 docker run --net=macvlan216 --ip=192.168.216.11 -it --rm debian
root@526f3060d759:/# ip a show eth0
94: eth0@if92: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether 8e:9a:99:25:b6:16 brd ff:ff:ff:ff:ff:ff
    inet 192.168.216.11/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::8c9a:99ff:fe25:b616/64 scope link tentative
       valid_lft forever preferred_lft forever
    inet6 fe94::2/64 scope link nodad
       valid_lft forever preferred_lft forever
       
root@526f3060d759:/# ip route
default via 192.168.216.1 dev eth0
192.168.216.0/24 dev eth0  proto kernel  scope link  src 192.168.216.11

root@526f3060d759:/# ip -6 route
fe80::/64 dev eth0  proto kernel  metric 256
fe94::/64 dev eth0  proto kernel  metric 256
default via fe94::10 dev eth0  metric 1024
```

### IPv6 Ipvlan L2 Mode


**Example:** IpVlan L2 Mode w/ 802.1q Vlan Tag:139, IPv6 Gateway:fe91::22

- Test: Start two containers on the same VLAN (139) and ping one another:

```
# Create a v6 network
docker network create -d ipvlan \
	--subnet=fe91::/64 --gateway=fe91::22 \
	-o parent=eth0.139 v6ipvlan139
	
# Start a container on the network
docker run --net=v6ipvlan139 -it --rm debian

```

View the container eth0 interface and v6 routing table:

```

74: eth0@if55: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether 00:50:56:2b:29:40 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe2b:2940/64 scope link
       valid_lft forever preferred_lft forever
    inet6 fe91::1/64 scope link nodad
       valid_lft forever preferred_lft forever
       
root@5c1dc74b1daa:/# ip -6 route
fe80::/64 dev eth0  proto kernel  metric 256
fe91::/64 dev eth0  proto kernel  metric 256
default via fe91::22 dev eth0  metric 1024
```

Start a second container and ping the first container's v6 address. 

```
$ docker run --net=v6ipvlan139 -it --rm debian

root@b817e42fcc54:/# ip a show eth0
75: eth0@if55: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether 00:50:56:2b:29:40 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe2b:2940/64 scope link tentative dadfailed
       valid_lft forever preferred_lft forever
    inet6 fe91::2/64 scope link nodad
       valid_lft forever preferred_lft forever
root@b817e42fcc54:/# ping6 fe91::1
PING fe91::1 (fe91::1): 56 data bytes
64 bytes from fe91::1%eth0: icmp_seq=0 ttl=64 time=0.044 ms
64 bytes from fe91::1%eth0: icmp_seq=1 ttl=64 time=0.058 ms

2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.044/0.051/0.058/0.000 ms
```

**Example:** Dual Stack IPv4/IPv6 with a VLAN ID:140

Next create a network with two IPv4 subnets and one IPv6 subnets, all of which have explicit gateways:

```
docker network  create  -d ipvlan \
	--subnet=192.168.140.0/24 --subnet=192.168.142.0/24 \
	--gateway=192.168.140.1  --gateway=192.168.142.1 \
	--subnet=fe99::/64 --gateway=fe99::22 \
	 -o parent=eth0.140 \
	 -o ipvlan_mode=l2 ipvlan140
```

Start a container and view eth0 and both v4 & v6 routing tables:

```
docker run --net=v6ipvlan139 --ip6=fe91::51 -it --rm debian

root@3cce0d3575f3:/# ip a show eth0
78: eth0@if77: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether 00:50:56:2b:29:40 brd ff:ff:ff:ff:ff:ff
    inet 192.168.140.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe2b:2940/64 scope link
       valid_lft forever preferred_lft forever
    inet6 fe99::1/64 scope link nodad
       valid_lft forever preferred_lft forever

root@3cce0d3575f3:/# ip route
default via 192.168.140.1 dev eth0
192.168.140.0/24 dev eth0  proto kernel  scope link  src 192.168.140.2

root@3cce0d3575f3:/# ip -6 route
fe80::/64 dev eth0  proto kernel  metric 256
fe99::/64 dev eth0  proto kernel  metric 256
default via fe99::22 dev eth0  metric 1024
```

Start a second container with a specific `--ip4` address and ping the first host using ipv4 packets:

```
docker run --net=ipvlan140 --ip=192.168.140.10 -it --rm debian
```

**Note**: Different subnets on the same parent interface in both Ipvlan `L2` mode and Macvlan `bridge` mode cannot ping one another. That requires a router to proxy-arp the requests with a secondary subnet. However, Ipvlan `L3` will route the unicast traffic between disparate subnets as long as they share the same `-o parent` parent link.



### IPv6 Ipvlan L3 Mode 


**Example:** IpVlan L3 Mode Dual Stack IPv4/IPv6, Multi-Subnet w/ 802.1q Vlan Tag:118

As in all of the examples, a tagged VLAN interface does not have to be used. The subinterfaces can be swapped with `eth0`, `eth1` or any other valid interface on the host other then the `lo` loopback.

The primary differnce you will see is that L3 mode does not create a default route with a next-hop but rather sets a default route pointing to `dev eth` only since ARP/Broadcasts/Multicase are all filtered by Linux as per the design.

```
# Create an IPv6+IPv4 Dual Stack Ipvlan L3 network 
# Gateways for both v4 and v6 are set to a dev e.g. 'default dev eth0'
docker network  create  -d ipvlan \
	--subnet=192.168.110.0/24 \
	--subnet=192.168.112.0/24 \
	--subnet=fe90::/64 \
	 -o parent=eth0.118 \
	 -o ipvlan_mode=l3 ipvlan118


# Start a few of containers on the network (ipvlan118) 
# Using Debian here because how busybox iproute2 
# handles unreachable network output is funky 
docker run --net=ipvlan118 -it --rm debian
# Start a second container specifying the v6 address
docker run --net=ipvlan118 --ip6=fe90::10 -it --rm debian
# Start a third specifying the IPv4 address
docker run --net=ipvlan118 --ip=192.168.112.50 -it --rm debian
# Start a 4th specifying both the IPv4 and IPv6 addresses
docker run --net=ipvlan118 --ip6=fe90::50 --ip=192.168.112.50 -it --rm debian
```

Interface and routing table outputs are as follows:

```
root@3a368b2a982e:/# ip a show eth0
63: eth0@if59: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether 00:50:56:2b:29:40 brd ff:ff:ff:ff:ff:ff
    inet 192.168.112.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fe2b:2940/64 scope link
       valid_lft forever preferred_lft forever
    inet6 fe90::10/64 scope link nodad
       valid_lft forever preferred_lft forever
       
root@3a368b2a982e:/# ip route
default dev eth0  scope link
192.168.112.0/24 dev eth0  proto kernel  scope link  src 192.168.112.2

root@3a368b2a982e:/# ip -6 route
fe80::/64 dev eth0  proto kernel  metric 256
fe90::/64 dev eth0  proto kernel  metric 256
default dev eth0  metric 1024
```

*Note:* There may be a bug when specifying `--ip6=` addresses when you delete a container with a specified v6 address and then start a new container with the same v6 address it throws the following like the address isn't properly being released to the v6 pool. It will fail to unmount the container and be left dead.

```
docker: Error response from daemon: Address already in use.
```


### Manually Creating 802.1q Links

**Vlan ID 40**

If you do not want the driver to create the vlan subinterface it simply needs to exist prior to the `docker network create`. If you have subinterface naming that is not `interface.vlan_id` it is honored in the `-o parent=` option again as long as the interface exists and us up.

Links if manually created can be named anything you want. As long as the exist when the network is created that is all that matters. Manually created links do not get deleted regardless of the name when the network is delted with `docker network rm`.

```
# create a new subinterface tied to dot1q vlan 40
ip link add link eth0 name eth0.40 type vlan id 40

# enable the new sub-interface
ip link set eth0.40 up

# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged
docker network  create  -d ipvlan \
   --subnet=192.168.40.0/24 \
   --gateway=192.168.40.1 \
   -o parent=eth0.40 ipvlan40

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=ipvlan40 -it --name ivlan_test5 --rm alpine /bin/sh
docker run --net=ipvlan40 -it --name ivlan_test6 --rm alpine /bin/sh
```

**Example:** Vlan subinterface manually created with any name:

```
# create a new subinterface tied to dot1q vlan 40
ip link add link eth0 name foobar type vlan id 40

# enable the new sub-interface
ip link set foobar up

# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged
docker network  create  -d ipvlan \
    --subnet=192.168.40.0/24 --gateway=192.168.40.1 \
    -o parent=foobar ipvlan40

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=ipvlan40 -it --name ivlan_test5 --rm alpine /bin/sh
docker run --net=ipvlan40 -it --name ivlan_test6 --rm alpine /bin/sh
```

Manually created links can be cleaned up with:

```
ip link del foobar
```


 A long list of examples can be found in the gist referenced at the beginning of the document here: [macvlan_ipvlan_docker_driver_manual_tests.txt](https://gist.github.com/nerdalert/9dcb14265a3aea336f40)