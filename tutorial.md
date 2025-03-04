Simple installation of OpenSDN vRouter Forwarder and usage
==========================================================

The goal of this document is to demonstrate how OpenSDN forwards
packets between virtual endpoints (virtual machines or containers),
how to install it in the minimal configuration and how to make it
working in very simple conditions.

The tutorial familiarizes a reader with basic concepts and definitions
employed in OpenSDN to transport packets in virtual networks.

Prerequisites
-------------

1. VirtualBox (7.1 seems to work with Ubuntu 22.04 host OS).
2. Ubuntu 22 OS running inside your VirtualBox or other environment.

A. Basic preparation steps
--------------------------

1. Install Ubuntu 22 OS (as a VM)
2. Install Docker Engine using the instruction from [https://docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu)
3. Update the software inside the VM
4. sudo docker pull ubuntu:jammy
5. sudo docker run --cap-add=NET_ADMIN --name cont1 -ti ubuntu:jammy bash:
    - apt update
    - apt install iproute2 -y
    - apt install iputils-ping
    - apt install netcat -y
6. sudo docker run --cap-add=NET_ADMIN --name cont2 -ti ubuntu:jammy bash:
    - apt update
    - apt install iproute2 -y
    - apt install iputils-ping
    - apt install netcat -y
7. sudo apt update

![Fig. A1: Starting the first container (cont1)](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-A-1.png)
*Fig. A1: Starting the first container (cont1)*

![Fig. A2: The network configuration of the first container (cont1)](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-A-2.png)
*Fig. A2: The network configuration of the first container (cont1)*

![Fig. A3: The list of images and containers running after the step A](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-A-3.png)
*Fig. A3: The list of images and containers running after the step A*

B. Installation of vRouter Forwarder and utilities
-----------------------------------------------

1. sudo apt install gcc make
2. sudo docker pull opensdn/tf-vrouter-kernel-build-init
3. install the corresponding kernel sources and headers (5.15), https://ubuntuhandbook.org/index.php/2023/11/install-ga-kernel-5-15-ubuntu-22-04/
4. remove other kernels, apt autoremove
5. instal vbox additions
6. sudo docker run --mount type=bind,src=/usr/src,dst=/usr/src --mount type=bind,src=/lib/modules,dst=/lib/modules
7. insmod vrouter ? ok dkms?. No, modprobe. But before, possibly dkms install vrouter is needed
8. download contrail-tools image: sudo docker pull opensdn/contrail-tools
9. sudo docker run --privileged --pid host --net host --name contrail-tools -ti opensdn/conrail-tools:latest
10. sudo apt install dkms -y

Constants can be found inside https://github.com/OpenSDN-io/tf-vrouter/blob/master/utils/pylib/constants.py

C. Configuration of containers interfaces
-----------------------------------------

D. Configuration of routing information
---------------------------------------

E. Final verification of connectivity between containers
--------------------------------------------------------

OpenSDN data plane basic defintions
-----------------------------------

Perhaps, this is the most important part of the document since it introduces
definitions which are mandatory for successfull administration, usage and
enhancement of OpenSDN virtual networks and applications.

So far we have encountered next data plane entities employed to organize 
communication between containers.

1. A **virtual interface** (or VIF) is a connected pair of interfaces with
one of them pointing into a virtual machine or a container and another one
attached to the host operating system where virtual resources (VMs or 
other type) are managed. Before using with vRouter Forwarder a virtual
interface must be plugged in using the **vif** utility or a Sandesh
request.
2. A **VRF table** provides a unique and isolated environment to store
routing (L3) and forwarding (L2) tables for switching packets between
virtual interfaces associated with a given identifier. The IP or MAC
addresses from two different VRF tables on a vRouter Forwarder can
intersect because of isolation principle.
3. A **route** is a record in a routing (L3) or a forwarding (L2) table having
the prefix (a MAC or IP address) that uniquely identify it and the pointer to
a **nexthop** intended for packets switching management in vRouter
Forwarder.
4. A **nexthop** defines an action performed on a packet with the header
matching the route pointing to this nexthop. In this case one says that
a packet hits the nexthop.
5. An **MPLS label** is an integer number used to identify a nexthop
directly instead of a route with the prefix.

Routes can be:
- bridge (AF_BRIDGE) when packets are transported using the
Ethernet header;
- IPv4 (AF_INET) when packets are transported using the IPv4 header;
- IPv6 (AF_INET6) when packets are transported using the IPv6 header;

Nexthops can have next popular types (according to the notation used in
nh utility):
- **drop**: which is also sometimes called "**discard**" is meant for dropping
packets hitting a nexthop of this type;
- **encap**: sends a packet hitting this nexthop to the virtual interface
associated with the nexthop;
- **tunnel**: sends a packet hitting this nexthop to the host with the underlay
IP address specified in the nexthop;
- **composite**: unites several nexthops (Encap or Tunnel) into a list to
organize multicast commutation (for L2 addressing) or L3 load balancing;
- VRF translate: transfers a packet from one VRF table into another.

Composite nexthops can have different sub-types:
- pure encap when all components of the composite nexthop list have type encap;
- pure tunnel when all components of the composite nexthop list have type tunnel;
- mixed type when components of the composite nexthop have different (encap or
tunnel) types.


Bibliography
------------

