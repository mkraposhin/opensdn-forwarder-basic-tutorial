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

The sketch of the setup is shown on Fig. P1.

A. Basic preparation steps
--------------------------

1. Install Ubuntu 22 OS (as a VM)
2. Install Docker Engine using the instruction from [https://docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu)
3. Update the software inside the VM:

        sudo apt update

4. Pull default Ubuntu image from dockerhub.io 

        sudo docker pull ubuntu:jammy

5. Run container number 1 (it will have name **cont1**) and install
necessary network utils inside it, see Fig. A1:

        sudo docker run --cap-add=NET_ADMIN --name cont1 -ti ubuntu:jammy bash:
        apt update
        apt install iproute2 -y
        apt install iputils-ping
        apt install netcat -y

6. Run container number 2 (it will have name **cont2**) and install
necessary network utils inside it, see Fig. A2:

        sudo docker run --cap-add=NET_ADMIN --name cont2 -ti ubuntu:jammy bash:
        apt update
        apt install iproute2 -y
        apt install iputils-ping
        apt install netcat -y

After all these actions we should have 2 Ubuntu 22 containers running with
names **cont1** and **cont2** inside our host operating system.

Finaly, its necessary to download the tutorial folders from GitHub into
the filesystem of the host OS running our containers **cont1** and **cont2**:

    git clone https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial.git tut-rep

![Fig. A1: Starting the first container (cont1)](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-A-1.png)
*Fig. A1: Starting the first container (cont1)*

![Fig. A2: The network configuration of the first container (cont1)](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-A-2.png)
*Fig. A2: The network configuration of the first container (cont1)*

![Fig. A3: The list of images and containers running after the step A](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-A-3.png)
*Fig. A3: The list of images and containers running after the step A*

B. Installation of vRouter Forwarder and utilities
-----------------------------------------------

1. Install gcc compiler in order to compile and install vRouter Forwarder:

        sudo apt install gcc make

2. Install the corresponding kernel sources and headers (5.15),
https://ubuntuhandbook.org/index.php/2023/11/install-ga-kernel-5-15-ubuntu-22-04/
and reboot your host OS.
3. Pull the image needed to build OpenSDN vRouter Forwarder from dockerhub:

        sudo docker pull opensdn/tf-vrouter-kernel-build-init

4. Remove other kernels from the host OS, for example using *apt autoremove*.
5. If host OS is running inside the VirtualBox, install vbox additions.
6. Reboot host OS.
7. Compile OpenSDN vROuter Forwader module by running the image downloaded
at step 3:

        sudo docker run --mount type=bind,src=/usr/src,dst=/usr/src --mount type=bind,src=/lib/modules,dst=/lib/modules

8. ??? sudo apt install dkms -y
8. Install the compiled vRouter Forwarder module into memory (No, modprobe. But before, possibly dkms install vrouter is needed????):

        modprobe vrouter

9. Download contrail-tools image:

        sudo docker pull opensdn/contrail-tools

9. Run contrail-tools in a separate terminal:

        sudo docker run --privileged --pid host --net host --name contrail-tools -ti opensdn/conrail-tools:latest



Constants can be found inside https://github.com/OpenSDN-io/tf-vrouter/blob/master/utils/pylib/constants.py

C. Configuration of containers interfaces
-----------------------------------------

Now we'll add virtual interfaces to allow communication between containers.
The default bridge interfaces already added by the docker system into the
containers are used for external communication with Internet, hence it is
better not to modify them.

A virtual interface pair is to be added to each container and then it
must be connected to vRouter Forwarder to enable packet switching between
these containers using OpenSDN. We'll use linux network namespaces to create
virtual interface pairs.

Since each container works in it's network namespace, we can create a linux
virtual interface pair (**veth**) and move one interface from these pair
into the container's network namespace. Thus we'll have network connectivity
between the container's and the host OS environments.

All configuration operations are stored inside 
[scripts](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/tree/main/scripts)
folder of the tutorial. The script [make-veth](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/scripts/make-veth)
creates new virtual pair, moves one virtual interface into the specified
container and assigns it the specified address:

    veth_name=$1 #veth1
    vethc_name=$2 #veth1c
    cont_name=$3 #cont1
    vethc_ip=$4
    
    cont_pid=`docker container inspect $cont_name --format '{{ .State.Pid }}'`
    echo "veth=$veth_name,vethc=$vethc_name,cont=$cont_name,pid=$cont_pid"
    ip link add $veth_name type veth peer name $vethc_name
    ip link set $vethc_name netns $cont_pid
    ip link set dev $veth_name up
    docker exec -ti $cont_name ip link set dev $vethc_name up
    docker exec -ti $cont_name ip addr add $vethc_ip dev $vethc_name

Firstly, we create a virtual pair for container **cont1**:

    sudo bash tut-rep/scripts/make-veth veth1 veth1c cont1 10.1.1.11/24

Then we create a virtual pair for container **cont2**:

    sudo bash tut-rep/scripts/make-veth veth2 veth2c cont1 10.1.1.12/24

Interfaces **veth1** and **veth2** reside in the host OS, while interfaces
**veth1c** and **veth2c** reside in the containers **cont1** and **cont2**
respectively.

Now we must tell OpenSDN vRouter Forwarder to catch packets from interfaces
**veth1** and **veth2** and switch them. This is achieved using **vif**
utility from OpenSDN contrail-tools package. The utils can be used to
list interfaces connected to OpenSDN vRouter Forwarder or it can be
use to connect host OS virtual or physical network interfaces. There
is a special script [make-vif](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/scripts/make-vif)
prepared for this tutorial to run this utility:

    veth_name=$1 #veth1
    veth_mac=00:00:5e:00:01:00
    cont_name=contrail-tools
    
    
    docker exec -ti $cont_name vif --add $veth_name --mac $veth_mac --vrf 0 --type virtual --transport virtual

It is assumed that the container with contrail-tools package is up and running
as **contrail-tools**. Next we type:

    sudo bash tut-rep/scripts/make-vif veth1
    sudo bash tut-rep/scripts/make-vif veth2

D. Configuration of routing information
---------------------------------------

Since we will need data from the tutorial's github repository, it is
recomended to clone it inside **contrail-tools** repository:

    git clone https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial.git tut-rep

OpenSDN vRouter Forwarder uses linux the Netlink protocol to receive
requests and send responses. Configuration requests can be written
in the form of XML files, then the Sandesh-based utility **vrcli** reads
an XML, decodes it's content encodes it into controlling messages and
sends them to vRouter Forwarder. All requests from this tutorial
are located in [xml_reqs](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_hugepages_conf.xml)
folder. **vrcli** can work both with vRouter running in DPDK or linux
kernel module modes. To make it working in the linux kernel module an
option **--vr_kmode** must be passed during an invocation of the utility.

Before working vRouter Forwarder, it must be instructed to initialize
important structure in memory, such as Forwarding Information Base (FIB)
tables, VRF tables, etc. Whereas vRouter Forwarder is able to work with
hugepages, the functionality is not used in this lab and memory is initialized
by passing empty list of files where memory is mapped. The corresponding
request is stored in [set_hugepages_conf.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_hugepages_conf.xml)
file.

We invoke the request from **contrail-tools** container using **vrcli**
command as follows:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_hugepages_conf.xml

We can check that the operation was succesfull by invoking:

    rt --dump ... --family bridge?

If one doesn't see error messages and sees instead empty routing tables,
then it means that vRouter Forwarder memory has been initialized
successfully, see Fig. B-1?

Next we must adjust the VRF table which is meant for keeping routing
information for packets travelling between **cont1** and **cont2**. In
OpenSDN dataplane, network traffic is switched between endpoints 
(VMs, containers, etc) of a hypervisor according to isolated routing
and forwarding tables united as a VRF table which has it's own identifier.
Every vRouter Forwarder can have up to 65536 VRF tables with identifiers
from 0 to 65535. And every virtual or physical interface connected to 
vRouter Forwarder and intended for overlay packets switching must be
linked with a VRF table.

We will use VRF table zero in this tutorial. To create this element in
vRouter Forwarder we'll use a request stored in
[set_vrf.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_vrf.xml)
file. The request (**vr_vrf_req**) is submitted to vRouter Forwarder as next:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_vrf.xml

We can verify that VRF table 0 is created using vrfdump utility:

    vrfdump

Before setting up routing information it's necessary to update configuration of
virtual interfaces in vRouter Forwarder. This is achieved by calling request
**vr_interface_req** stored in files [set_vif1_ip.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_vif1_ip.xml) and [set_vif2_ip.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_vif2_ip.xml) for containers **cont1** and **cont2** respectively.

However, these requests must be modified before their invocation. Namely:
- field <vifr_idx></vifr_idx> must contain actual index of an interface
from the host OS;
- field <vifr_ip></vifr_ip> must contain IPv4 address of an interface;
- field <vifr_nh_id></vifr_nh_id> must contain identifier of a nexthop
attached to this interface.

How to get vifr_id?

How to set set vifr_ip?

For the nexthop identifier (or vifr_nh_id field) we must agree here,
because we do not have any nexthops in vRouter Forwarder now. Let's
take nexthop number 1 for virtual interface 1 (**veth1**) and nexthop 2 for
virtual interface 2 (**veth2**).

Then we invoke our requests:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_vif1_ip.xml
    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_vif2_ip.xml


E. Fi nal verification of connectivity between containers
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

