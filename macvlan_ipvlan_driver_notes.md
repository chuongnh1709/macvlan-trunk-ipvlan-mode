
# Macvlan, Ipvlan and 802.1q Trunk Driver Notes

### Contents

 - [MacVlan Bridge Mode Example Usage](#macvlan-bridge-mode-example-usage)

 - [Ipvlan L2 Mode Example Usage](#ipvlan-l2-mode-example-usage)

 - [Macvlan 802.1q Trunk Bridge Mode Example Usage](#macvlan-8021q-trunk-bridge-mode-example-usage)

 - [Ipvlan 802.1q Trunk L2 Mode Example Usage](#ipvlan-8021q-trunk-l2-mode-example-usage)

 - [IPVlan L3 Mode Example Usage](#ipvlan-l3-mode-example-usage)

 - [IPv6 Example Usage](#ipv6)

 - [Manual 802.1q Link Creation Example](#manually-creating-8021q-links)


Branch at: [https://github.com/nerdalert/libnetwork/tree/ipvlan_macvlan](https://github.com/nerdalert/libnetwork/tree/ipvlan_macvlan). This branch contains both macvlan and ipvlan as the mechanics are structure are very similar.


- The driver caches `NetworkCreate` callbacks to the boltdb datastore along with populating `*networks`. In the case of a restart, the driver initializes the datastore with `Init()` and populates `*networks` since `NetworkCreate()` is only called once. 
- There can only be one (ipvlan or macvlan) driver type bound to a host interface with running containers at any given time. Currently the driver does not prevent ipvlan and macvlan networks to be created with the same `--host_interface` but will throw an error if you try to start an ipvlan container and a macvlan container at the same time on the same `--host_interface`. A mix of host interfaces and macvlan/ipvlan types can be used with running containers, each interface just needs to use the same type. Example: Macvlan Bridge or IPVlan L2. There is no mixing of running containers on the same host interface. 
- The specified gateway is external to the host or at least not defined by the driver itself. 
- Each network is isolated from one another. Any container inside the network/subnet can talk to one another without a reachable gateway in both `macvlan bridge` mode and `ipvlan L2` mode. IP tables may be able to work around that if a user wanted to.
- Containers on separate networks cannot reach one another without an external process routing between the two networks/subnets.
- **Note:** In both Macvlan and Ipvlan you are not able to ping or communicate with the default namespace IP address. For example, if you create a container and try to ping the Docker host's `eth0` it will **not** work. That traffic is explicitly filtered by the kernel modules themselves to offer additional provider isolation and security.

- More information about Ipvlan & Macvlan can be found in the upstream [readme](https://github.com/torvalds/linux/blob/master/Documentation/networking/ipvlan.txt)
 

- If you just want to test the Docker binary with the drivers compiled in download: [docker-ipvlan-macvlan-binary.tar](https://www.dropbox.com/s/tsz9ktobg4dkwuy/docker-1.11.0-dev.zip?dl=1) 

- For a list of manual tests you can paste in and give a whirl see:  [macvlan_ipvlan_docker_driver_manual_tests.txt](https://gist.github.com/nerdalert/9dcb14265a3aea336f40)

- The Docker build is the following:

```
docker -v
Docker version 1.11.0-dev, build ac9d1b7-unsupported
```

### MacVlan Bridge Mode Example Usage ###

- In this example, `eth0` on the docker host has an IP on the `192.168.1.0/24` network and a default gateway of `192.168.1.1`. The gateway is an external router with an address of `192.168.1.1`. An IP address is not required on the Docker host interface `eth0` in `bridge` mode, it merely needs to be on the proper upstream network to get forwarded by a network switch or network router.

<a href="url"><img src="https://cloud.githubusercontent.com/assets/1711674/13232710/5c1c7d2a-d97e-11e5-99e9-f19d1aa0f87f.jpg" align="center" width="350" ></a>

**Note** For Macvlan bridge mode and Ipvlan L2 mode the subnet values need to match the NIC's interface of the Docker host. For example, Use the same subnet and gateway of the Docker host ethernet interface that is specified by the `-o host_iface=` option.

- The parent interface used in this example is `eth0` and it is on the subnet `192.168.1.0/24`. The containers in the `docker network` will also need to be on this same subnet as the parent `--host_iface=`. The gateway is an external router on the network, not any ip masquerading or any other local proxy (see [diagrams for topologies](https://cloud.githubusercontent.com/assets/1711674/13032315/099ddbb2-d2bb-11e5-86c1-a526ade612cc.jpg)).

- The driver is specified with `-d driver_name` option. In this case `-d macvlan`

- The parent interface `-o host_iface=eth0` is configured as followed:

```
ip addr show eth0
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.1.251/24 brd 192.168.1.255 scope global eth0
```

Create the macvlan network and run a container attaching to it:

```
# Macvlan  (--macvlan_mode= Defaults to Bridge mode if not specified)
docker network  create  -d macvlan  --subnet=192.168.1.0/24 --gateway=192.168.1.1 -o host_iface=eth0 macnet100
docker  run --net=macnet100 -it --rm busybox
```

You can explicitly specify the `bridge` mode option `-o macvlan_mode=bridge`. It is the default so will be in `bridge` mode either way.



### Ipvlan L2 Mode Example Usage ###

The ipvlan `L2` mode example is virtually identical to the macvlan `bridge` mode example. The driver is specified with `-d driver_name` option. In this case `-d ipvlan`

<a href="url"><img src="https://cloud.githubusercontent.com/assets/1711674/13232749/7999fdf0-d97e-11e5-9e62-8793ccdcb061.jpg" align="center" width="350" ></a>


The parent interface `-o host_iface=eth0` is configured as followed:

```
ip addr show eth0
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    inet 192.168.1.251/24 brd 192.168.1.255 scope global eth0
```

Create the ipvlan network and run a container attaching to it:

```
# Ipvlan  (--ipvlan_mode= Defaults to L2 mode if not specified)
docker network  create -d ipvlan  --subnet=192.168.1.0/24 --gateway=192.168.1.1 -o host_iface=eth0 ipnet100
docker  run --net=ipnet100 -it --rm busybox
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

In the first network tagged and isolated by the Docker host, `eth0.50` is the parent interface tagged with vlan id `50` specified with `-o host_iface=eth0.50`. Other naming formats can be used, but the links need to be added and deleted [manually](https://gist.github.com/nerdalert/28168b016112b7c13040#manually-creating-8021q-links) using `ip link` or Linux configuration files. As long as the `--host_iface` exists anything can be used if compliant with Linux netlink.

```
# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged
docker network  create  -d macvlan  --subnet=192.168.50.0/24 --gateway=192.168.50.1 -o host_iface=eth0.50 macvlan50

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=macvlan50 -it --name macvlan_test5 --rm busybox
docker run --net=macvlan50 -it --name macvlan_test6 --rm busybox
```

**Vlan ID 60**

In the second network, tagged and isolated by the Docker host, `eth0.60` is the parent interface tagged with vlan id `60` specified with `-o host_iface=eth0.60`. The `macvlan_mode=` defaults to `macvlan_mode=bridge`. It can also be explicitly set with the same result as shown in the next example.

```
# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged. 
docker network  create  -d macvlan  --subnet=192.168.60.0/24 --gateway=192.168.60.1 -o host_iface=eth0.60 -o --macvlan_mode=bridge macvlan60

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=macvlan60 -it --name macvlan_test7 --rm ubuntu
docker run --net=macvlan60 -it --name macvlan_test8 --rm ubuntu
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

In the first network tagged and isolated by the Docker host, `eth0.20` is the parent interface tagged with vlan id `20` specified with `-o host_iface=eth0.20`. Other naming formats can be used, but the links need to be added and deleted manually using `ip link` or Linux configuration files. As long as the `--host_iface` exists anything can be used if compliant with Linux netlink.

```
# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged
docker network  create  -d ipvlan  --subnet=192.168.20.0/24 --gateway=192.168.20.1 -o host_iface=eth0.20 ipvlan20

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=ipvlan20 -it --name ivlan_test1 --rm busybox
docker run --net=ipvlan20 -it --name ivlan_test2 --rm busybox
```

**Vlan ID 30**

In the second network, tagged and isolated by the Docker host, `eth0.30` is the parent interface tagged with vlan id `30` specified with `-o host_iface=eth0.30`. The `ipvlan_mode=` defaults to l2 mode `ipvlan_mode=l2`. It can also be explicitly set with the same result as shown in the next example.

```
# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged. 
docker network  create  -d ipvlan  --subnet=192.168.30.0/24 --gateway=192.168.30.1 -o host_iface=eth0.30 --ipvlan_mode=l2 ipvlan30

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=ipvlan30 -it --name ivlan_test3 --rm ubuntu
docker run --net=ipvlan30 -it --name ivlan_test4 --rm ubuntu
```

The gateway is set inside of the container as the default gateway. That gateway would typically be an external router on the network.

```
$ ip route
default via 192.168.30.1 dev eth0
192.168.30.0/24 dev eth0  src 192.168.30.2
```



### IPVlan L3 Mode Example Usage ###

-Ipvlan L3 mode drops all broadcast and multicast traffic. 
-L3 mode needs to be on a separate subnet as the default namespace since it requires a netlink route in the default namespace pointing to the Ipvlan parent interface. 
- The parent interface used in this example is `eth0` and it is on the subnet `192.168.1.0/24`. Notice the `docker network` is **not** on the same subnet as `eth0`.
- Unlike macvlan bridge mode and ipvlan l2 modes, different subnets/networks can ping one another as long as they share the same parent interface `--host_interface=`.

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
docker network  create -d ipvlan --subnet=192.168.120.0/24 -o host_iface=eth0 -o ipvlan_mode=l3 ipvlan120
docker run --net=ipvlan120 -it --rm busybox 

# Or specify a specific IPv4 address
docker run --net=ipvlan120 --ip=192.168.120.10 -it --rm busybox 
```

You will notice there is no `--gateway=` option in the network create. The field is ignored if one is specified since it doesn't have a use in `l3` mode. Take a look at the container routing table from inside of the container:

```
# Inside the container
$ ip route
default dev eth0
192.168.120.0/24 dev eth0  src 192.168.120.2
```


-Example: This route is added to the Docker host not the Docker container. `eth0` in the route has to be the same interface as specified in the `-o host_iface=eth0` network create.

In order to ping the container from a remote host or the container be able to ping a remote host, the remote host needs to have a route pointing to the host IP address of the container's Docker host eth interface. 

-If you have two hosts and one of them has a ipvlan l3 mode container running, the second host needs to have a route added to ping the container. The following are the addresses of the example of two hosts, host1 running docker with a container in ipvlan l3 mode and *host2* a regular host we will be pinging from the container and vice versa:

-*Host1:* Docker Host eth0: 192.168.1.249/24
-*Host1:* Docker ipvlan l3 mode container network: 192.168.130.0/24
-*Host1:* Docker ipvlan l3 mode container IP: 192.168.130.2
-*Host2:* Remote Host eth0: 192.168.1.251/24

First, add a route to the remote host that tells the host, in order to reach the container network `192.168.130.0/24` on *Host1* use the next hop *Host1* eth0 `192.168.1.249`. By adding the following route to

```
# On Host2 add a route for the L3 container network to be reached via the Docker host eth0 address (192.168.1.249).
ip route add 192.168.130.0/24 via 192.168.1.249
```

After adding the above route to *Host1* it should be able to ping the L3 mode container at `192.168.130.2` from its eth0 interface with the IP `192.168.1.251` as should the container be able to ping `192.168.1.249` from its address at `192.168.130.2`.

L3 (layer 3) or ip routing has the ability to scale well beyond L2 (layer 2) networks. The Internet is made up of a collection of interconnected L3 networks. This is attractive when coupled with the density presented by migrations to Docker containers and worth spending some time to understand if new to scaling networks.


### IPv6 

- Default to dual stack? Currently does with generic v4 and gateway services. If ipv4 is not specified should we disable gateway services and provision v6 only etc.
- Should `EnableIPv6` be used? 

Example: start two containers and ping one another:

```
# Create a v6 network
docker network create -d macvlan  --subnet=fe90::/64 --gateway=fe90::22 -o host_iface=eth0 macnetv6
# Start a container on the network
docker run --net=macnetv6 -it --rm busybox

```

View the container eth0 interface:

```
# IP config of the container once it starts:

$ ip addr show eth0
eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether ce:26:6b:b8:27:11 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::cc26:6bff:feb8:2711/64 scope link tentative
       valid_lft forever preferred_lft forever
    inet6 fe90::1/64 scope link flags 02
       valid_lft forever preferred_lft forever
```

Start a second container and ping the first containers v6 address at `fe90::1` from it's address of `fe90::2`.

```
$ ip a show eth0
48: eth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 06:09:43:a2:91:c3 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::409:43ff:fea2:91c3/64 scope link
       valid_lft forever preferred_lft forever
    inet6 fe90::2/64 scope link flags 02
       valid_lft forever preferred_lft forever

$ ping -6 fe90::1
PING fe90::2 (fe90::1): 56 data bytes
64 bytes from fe90::1: seq=0 ttl=64 time=0.044 ms
```



### Manually Creating 802.1q Links

**Vlan ID 40**

If you do not want the driver to create the vlan subinterface it simply needs to exist prior to the `docker network create`. 

```
# create a new subinterface tied to dot1q vlan 40
ip link add link eth0 name eth0.40 type vlan id 40

# enable the new sub-interface
ip link set eth0.40 up

# now add networks and hosts as you would normally by attaching to the master (sub)interface that is tagged
docker network  create  -d ipvlan  --subnet=192.168.40.0/24 --gateway=192.168.40.1 -o host_iface=eth0.40 ipvlan40

# in two separate terminals, start a Docker container and the containers can now ping one another.
docker run --net=ipvlan40 -it --name ivlan_test5 --rm ubuntu
docker run --net=ipvlan40 -it --name ivlan_test6 --rm ubuntu
```