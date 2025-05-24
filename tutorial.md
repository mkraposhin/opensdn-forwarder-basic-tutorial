Minimal installation of OpenSDN vRouter Forwarder and it's usage
================================================================

The goal of this document is to demonstrate how OpenSDN forwards
packets between virtual endpoints (virtual machines or containers),
how to install it in the minimal configuration and how to make it
working in very simple conditions.

The tutorial familiarizes a reader with basic concepts and definitions
employed in OpenSDN to transport packets in virtual networks.

Prerequisites
-------------

1. Ubuntu 22 OS running inside your VirtualBox or other environment.
2. Root access to this OS.

Introduction
------------

The material demonstrates basics of network communication using OpenSDN
vRouter Forwarder and allows a reader to understand deeper basic concepts
which are used in OpenSDN dataplane and how this component is programmed
by higher level components (by the vRouter Agent, the Controller and,
eventually, the Config).

Essentially, the steps covered in this tutorial reproduce requests which are
sent by vRouter Agent to vRouter Forwarder when a virtual network configuration
is modified using Config API or OpenSDN UI mechanisms. The understanding of
how vRouter Forwarder is programmed by higher level components should
facilitate rational introduction of changes in an OpenSDN virtual network
configuration and analysis of arising faults and their reasons.

The network configuration to consider was intentionally selected very simple.
The sketch of the network setup is shown on Fig. I1: we have 2 endpoints,
each having an IP address (10.1.1.11/24 and 10.1.1.22/24) and communicating
with each other via a switch which is imitated using OpenSDN vRouter Forwarder.

From the operating system point of view we will have 2 containers connected
to the host OS and to the vRouter Forwarder via veth pairs (
**veth1**/**veth1c** and **veth2**/**veth2c**), Fig. I2.

![Fig. I1: The considered virtual network configuration design](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-I-1.jpg)
*Fig. I1: The considered virtual network configuration design*

![Fig. I2: The virtual network configuration implementation](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-I-2.jpg)
*Fig. I2: The virtual network configuration implementation*

A. Basic preparation steps
--------------------------

1. Install Ubuntu 22 OS (as a VM)
2. Install Docker Engine using the instruction from 
[https://docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu)
3. Update the software inside the VM:

        sudo apt update

4. Pull default Ubuntu image from dockerhub.io 

        sudo docker pull ubuntu:jammy

5. Run container number 1 (it will have name **cont1**) and install
necessary network utils inside it, see Fig. A1:

        sudo docker run --cap-add=NET_ADMIN --name cont1 -ti ubuntu:jammy bash
        apt update
        apt install iproute2 iputils-ping netcat git nano vim-tiny -y

6. Run container number 2 (it will have name **cont2**) and install
necessary network utils inside it, see Fig. A2:

        sudo docker run --cap-add=NET_ADMIN --name cont2 -ti ubuntu:jammy bash
        apt update
        apt install iproute2 iputils-ping netcat git nano vim-tiny -y

After all these actions we must have 2 Ubuntu 22 containers running with
names **cont1** and **cont2** inside the host operating system.

Finally, its necessary to download the tutorial folders from GitHub repository
into the filesystems of the **host OS / Ubuntu OS**, containers **cont1** and **cont2**:

    git clone https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial.git tut-rep

It is assumed that the repository is downloaded inside the user home directory of
the host OS and inside the root directory (/) of containers **cont1**
and **cont2**.

![Fig. A1: Starting the first container (cont1)](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-A-1.png)
*Fig. A1: Starting the first container (cont1)*

![Fig. A2: The network configuration of the first container (cont1)](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-A-2.png)
*Fig. A2: The network configuration of the first container (cont1)*

![Fig. A3: The list of images and containers running after the step A](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-A-3.png)
*Fig. A3: The list of images and containers running after the step A*

B. Installation of vRouter Forwarder and utilities
--------------------------------------------------

The technical steps to build and run vRouter Forwarder manually are
covered in details here since usually this process is completed automatically
using some installation scripts, which are not used because this tutorial
is claimed as minimal.

1. Install gcc compiler and other tools in order to compile and install vRouter
Forwarder in the host OS:

        sudo apt install gcc make dkms -y

2. Install the corresponding kernel sources and headers (5.15),
https://ubuntuhandbook.org/index.php/2023/11/install-ga-kernel-5-15-ubuntu-22-04/
and reboot your host OS.
3. Pull the image needed to build in the host OS OpenSDN vRouter Forwarder
from dockerhub:

        sudo docker pull opensdn/tf-vrouter-kernel-build-init

4. Remove other kernels from the host OS, for example using *apt autoremove*.
5. If the host OS is running inside the VirtualBox, install vbox additions.
6. Reboot the host OS to boot into the installed 5.15 kernel.
7. Compile the OpenSDN vRouter Forwarder module by running the image downloaded
at step 3 in the host OS:

        sudo docker run --mount type=bind,src=/usr/src,dst=/usr/src --mount type=bind,src=/lib/modules,dst=/lib/modules opensdn/tf-vrouter-kernel-build-init:latest

8. The build and installation process should output information about the progress on the screen. The file vrouter.ko must appear inside /lib/modules/$(uname -r)/updates/dkms of the host OS filesystem (see Fig. B1). Install the compiled vRouter Forwarder module into memory using modprobe
in the host OS:

        sudo modprobe vrouter

9. Check that **vrouter** module has been installed in the host OS:

        lsmod | grep vrouter

10. Download contrail-tools image in the host OS:

        sudo docker pull opensdn/contrail-tools

11. Run contrail-tools in a separate terminal of the host OS:

        sudo docker run --privileged --pid host --net host --name contrail-tools -ti opensdn/contrail-tools:latest

12. Copy the tutorial repository folder from the **host OS** into the
root (/) folder of **contrail-tools** container

        tar cfz tut-rep.tgz tut-rep && sudo docker cp ./tut-rep.tgz contrail-tools:/tut-rep.tgz && sudo docker exec -ti contrail-tools tar xfz tut-rep.tgz

14. Run containers **cont1** and **cont2** (in separate terminal windows or
tabs) in the host OS:

        sudo docker start cont1
        sudo docker start cont2

    and then start the bash interpreter in separate terminals:
    - for **cont1**:

            sudo docker exec -ti cont1 bash

    - for **cont2**:

            sudo docker exec -ti cont2 bash

![Fig. B1: An example of the vRouter Forwader build process output](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-B-1.png)
*Fig. B1: An example of the vRouter Forwader build process output*

C. Configuration of containers interfaces
-----------------------------------------

Now we'll add virtual interfaces to allow communication between containers.
The default bridge interfaces that were already added by the docker system into the
containers are used for external communication with Internet, hence it is
better not to modify them.

Therefore, an additional virtual interface pair is to be added to each container
and then connected to vRouter Forwarder to enable packet transfer between
containers **cont1** and **cont2** using OpenSDN. Linux network namespaces
will be used to create these virtual interface pairs.

Since each container works in it's network namespace, we can create a linux
virtual interface pair (**veth**) and move one interface from this pair
into the container's network namespace. Thus we'll have network connectivity
between the container's and the host OS's environments.

All configuration operations are stored inside 
[scripts](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/tree/main/scripts)
folder of the tutorial. The script [make-veth](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/scripts/make-veth)
creates a new virtual pair, moves one virtual interface into the specified
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

Firstly, we create a virtual pair for container **cont1** on the host OS:

    sudo bash tut-rep/scripts/make-veth veth1 veth1c cont1 10.1.1.11/24

Then we create a virtual pair for container **cont2**:

    sudo bash tut-rep/scripts/make-veth veth2 veth2c cont2 10.1.1.22/24

After these steps we must see 2 new interfaces in the host OS (Fig. C1)
and one new interface in **cont1** and **cont** (Fig. C2 and Fig. C3
respectively).
Interfaces **veth1** and **veth2** reside in the host OS, while interfaces
**veth1c** and **veth2c** reside in the containers **cont1** and **cont2**
respectively.

Now we must tell OpenSDN vRouter Forwarder to catch packets from interfaces
**veth1** and **veth2** and transfer them. This is achieved using **vif**
utility from OpenSDN contrail-tools package. The utility can be used to
list interfaces connected to OpenSDN vRouter Forwarder or it can be also
used to connect host OS virtual or physical network interfaces. There
is a special script 
[make-vif](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/scripts/make-vif)
prepared for this tutorial to run this utility:

    veth_name=$1 #veth1
    veth_mac=00:00:5e:00:01:00
    cont_name=contrail-tools
    
    
    docker exec -ti $cont_name vif --add $veth_name --mac $veth_mac --vrf 0 --type virtual --transport virtual

It is assumed that the container with contrail-tools package is up and running
under the name **contrail-tools**. If so, then one types in the host OS:

    sudo bash tut-rep/scripts/make-vif veth1
    sudo bash tut-rep/scripts/make-vif veth2

The result of this operation can be verified using **vif** utility inside
**contrail-tools** container:

    vif --list

The utility returns the list of the atthached to vRouter Forwarder interfaces
among which we should see records for **veth1** and **veth2**, Fig. C4.

![Fig. C1: An example of "ip a" output in the host OS](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-C-1.png)
*Fig. C1: An example of "ip a" output in the host OS*

![Fig. C2: An example of "ip a" output in cont1](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-C-2.png)
*Fig. C2: An example of "ip a" output in **cont1***

![Fig. C3: An example of "ip a" output in cont2](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-C-3.png)
*Fig. C3: An example of "ip a" output in **cont2***

![Fig. C4: An example of "vif --list" output in contrail-tools](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-C-4.png)
*Fig. C4: An example of "vif --list" output in **contrail-tools***

D. Configuration of routing information
---------------------------------------

OpenSDN vRouter Forwarder uses the Netlink protocol to receive
requests and send responses. Configuration requests can be written
in the form of XML files, then the Sandesh-based utility **vrcli** reads
an XML, decodes it's content, encodes it into controlling messages and
sends them to vRouter Forwarder. All requests from this tutorial
are located in [xml_reqs](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_hugepages_conf.xml)
folder. **vrcli** can work both with vRouter running in DPDK or linux
kernel module modes. To make it working in the linux kernel module an
option **--vr_kmode** must be passed during the invocation of the utility.

Before working with vRouter Forwarder, it must be instructed to initialize
important structures in memory, such as Forwarding Information Base (FIB)
tables, VRF tables, etc. Whereas vRouter Forwarder is able to work with
hugepages, the functionality is not used in this lab and memory is initialized
by passing empty list of files where memory is mapped. The corresponding
request is stored in [set_hugepages_conf.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_hugepages_conf.xml)
file.

We invoke the request from **contrail-tools** container using **vrcli**
command as follows (in the directory where the tutorial repository was
cloned into):

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_hugepages_conf.xml

As a response we must see:

    Running Sandesh request...

Usually this means that the request producing such message was accepted and 
processed. We can check that the operation was succesfull by invoking the
next command inside **contrail-tools** container:

    rt --dump 0 --family bridge

If one doesn't see error messages and sees instead empty routing tables,
then it means that vRouter Forwarder memory has been initialized
successfully, see Fig. D-1.

![Fig. D1: An example of "rt" output in contrail-tools](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-D-1.png)
*Fig. D1: An example of "rt" output in contrail-tools*

Next we must adjust the VRF table which is meant for keeping routing
information for packets travelling between **cont1** and **cont2**. In
OpenSDN dataplane, network traffic is switched between endpoints 
(VMs, containers, etc) of a hypervisor according to isolated routing
and forwarding tables stored as a VRF table which has it's own identifier.
Every vRouter Forwarder can have up to 65536 VRF tables with identifiers
from 0 to 65535. And every virtual or physical interface connected to 
vRouter Forwarder (and intended for overlay packets switching) must be
linked with a VRF table.

We will use VRF table zero (0) in this tutorial. To create this element in
vRouter Forwarder we'll use a request stored in
[set_vrf.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_vrf.xml)
file. The request (**vr_vrf_req**) is submitted to vRouter Forwarder as follows:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_vrf.xml

We can verify that VRF table 0 has been created using **vrftable** utility:

    vrftable --dump

Before setting up routing information it's necessary to update configuration of
virtual interfaces in vRouter Forwarder. This is achieved by calling a request
**vr_interface_req** stored in files [set_vif1_ip.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_vif1_ip.xml) and [set_vif2_ip.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_vif2_ip.xml) for containers **cont1** and **cont2** respectively.

However, these requests must be modified before their invocation. Namely:
- field <vifr_idx></vifr_idx> must contain the actual index of an interface
from the host OS;
- field <vifr_ip></vifr_ip> must contain the IPv4 address of an interface;
- field <vifr_nh_id></vifr_nh_id> must contain the identifier of the nexthop
attached to this interface.

We take **vifr_idx** from from the output of **ip a** command: it is a number
before a colon preceding the name of the corresponding interface (**veth1**
or **veth2**), see Fig. C1.

The value of IPv4 address is submitted into **vr_interface_req** as a
4-byte integer number, therefore, we must convert IP addresses from the
network configuration introduced earlier:
- 10.1.1.11 converts into 11\*256\*256\*256 + 1\*256\*256 + 1*256 + 10 or
184615178;
- 10.1.1.22 converts into
22\*256\*256\*256 + 1\*256\*256 + 1*256 + 10 or 369164554.

For the nexthop identifier (**vifr_nh_id** field) we must agree here,
because we do not have any nexthops in vRouter Forwarder now and these
numbers are arbitrary (i.e. they don't need to follow any rules). Let's
choose nexthop number 1 for virtual interface 1 (**veth1**) and nexthop 2 for
virtual interface 2 (**veth2**).

Then we invoke our requests in **contrail-tools** container:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_vif1_ip.xml
    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_vif2_ip.xml

If requests are accepted by OpenSDN vRouter Forwarder, we must see again:

    Running Sandesh request...

Then we can **vif** utility to verify our interface configuration in OpenSDN
vRouter Forwarder (see Fig D-2 for example):

    vif --list

![Fig. D2: An example of "vif" output in contrail-tools after the addition of interfaces](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-D-2.png)
*Fig. D2: An example of "vif" output in contrail-tools after the addition of interfaces*

Nexthops 1 and 2 are referenced by our interfaces, but they do not present in
our configuration and this can be checked with the utility **nh**:

    nh --list

To add a nexthop into OpenSDN vRouter Forwarder table one uses **vr_nexthop_req**
requests. Examples of these requests for nexthops associated with interfaces
**veth1** and **veth2** are stored in files
[set_cont1_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_cont1_br_nh.xml)
and
[set_cont2_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_cont2_br_nh.xml)
respectively.

To adjust these templates for the local configuration next fields must be corrected:
- <nhr_encap_oif_id></nhr_encap_oif_id> storing labels of interfaces **veth1** 
or **veth2** associated with the given nexthop;
- <nhr_encap></nhr_encap> storing local (the container or the VM) MAC address of
the interface with we associate the nexthop.

For the interface label (ID) we replace value "0":

    <element>0</element>
with the actual ID of **veth1** or **veth2** interface from the output of
**vif** command (**contrail-tools** container), see Fig. D2 for example:

    vif --list

The array storing **nhr_encap** field contains six elements representing bytes
of an associated virtual interface MAC address (**veth1c** or **veth2c**
in our case). These elements are marked with numbers 1 to 6 in the templates:

          <element>1</element>
          <element>2</element>
          <element>3</element>
          <element>4</element>
          <element>5</element>
          <element>6</element>

There is a script
[devmac2list](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/scripts/devmac2list)
which simplifies retrieval of these elements from an interface (in order to
use it, the repository must be cloned into **cont1** and **cont2**).
If the repository was cloned into folder **/tut-rep** of **cont1** and
**cont2** containers, then we can run next commands from the host OS:

    sudo docker exec -ti cont1 bash /tut-rep/scripts/devmac2list veth1c
    sudo docker exec -ti cont2 bash /tut-rep/scripts/devmac2list veth2c

The output of these commands must be similar to Fig. D-3. Note, that here we
use **veth1c** and **veth2c** instead of **veth1** and **veth2** because
we need MAC addresses of the interfaces from containers, not from the host OS.
Also, whereas we take values from **cont1** and **cont2** containers, we
modify XML files with requests inside **contrail-tools** container.

The obtained values can be copied into requests stored in
**set_cont1_br_nh_req.xml** and **set_cont2_br_nh_req.xml** files in 
**contrail-tools** container. Then we run these requests in order to add
nexthops into vRouter Forwarder (**contrail-tools** container):

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_cont1_br_nh.xml
    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_cont2_br_nh.xml

Now we can check that nexthops were added by typing in **contrail-tools**
container:

    nh --list

The output of the command above must be similar to the Fig. D-4. From the 
output we can verify that the correct interfaces were associated with our
nexthops and we can also compare MAC strings in nexthops 1 and 2 with
**veth1c** and **veth2c** MAC addresses (can be obtained with **ip**
command inside the corresponding containers).

Many vRouter Forwarder creation or modification requests also use different
flags and options to specify operation regimes. The values of these flags
can be found inside 
[https://github.com/OpenSDN-io/tf-vrouter/blob/master/utils/pylib/constants.py](https://github.com/OpenSDN-io/tf-vrouter/blob/master/utils/pylib/constants.py)

![Fig. D3: The output of "devmac2list" showing MAC address of interfaces veth1c and veth2c as lists for an XML request](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-D-3.png)
*Fig. D3: The output of "devmac2list" showing MAC address of interfaces veth1c and veth2c as lists for an XML request*

![Fig. D4: An example of "nh" output in contrail-tools with list of L2 nexthops pointing to interfaces veth1c and veth2c](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-D-4.png)
*Fig. D4: An example of "nh" output in contrail-tools with list of L2 nexthops pointing to interfaces veth1c and veth2c*

While our interfaces are now associated with nexthops, vRouter Forwarder
still doesn't have enough information to transfer packets between **veth1c**
and **veth2c** because there is no association between a packet header
and an interface accepting the packet. This gap is filled with
forwarding tables. Firstly, we need a bridge (L2) forwarding table
records (or route records) to enable transport of Ethernet frames
through vRouter Forwarder. A route record is basically a structure
containing a prefix address (MAC, IPv4 or IPv6) and the link to the
nexthop defining what to do next with the packet whose outer header matches
the route prefix address.

Before adding a route record we have to associate our nexthops
with MPLS labels since OpenSDN is MPLS-based SDN and it requires
this type of addressing for bridge communications. MPLS labels
are introducted using **vr_mpls_req** requests. The example request
is stored in
[set_mpls1.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_mpls1.xml)
and
[set_mpls2.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_mpls2.xml)
file. As it is seen from the content of the file, the request is quite simple:
it just associates a nexthop number with an MPLS label number.

In order to create MPLS labels for our nexthops 1 and 2 it's necessary
to run the request **set_mpls.xml** file from **contrail-tools**
container:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_mpls1.xml
    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_mpls2.xml

The results of the command can be verified with **mpls** utility in
**contrail-tools** container (see Fig. D-5 for example):

    mpls --dump

![Fig. D5: An example of "mpls" output in contrail-tools with the list of new MPLS labels](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-D-5.png)
*Fig. D5: An example of "mpls" output in contrail-tools with the list of new MPLS labels*

L2 route records are added via **vr_route_req** requests. The request
must contain: 
- a VRF table ID (0 in our example) defined in the field <rtr_vrf_id></rtr_vrf_id>;
- a family type (7 which is basically the value of AF_BRIDGE constant) defined
in the field <rtr_family></rtr_family>
- the label of a nexthop (1 or 2 for these L2 routes) defined in <rtr_nh_id></rtr_nh_id>;
- and a prefix which is determined by a MAC address of
a container or a VM interface (**veth1c** or **veth2c** in this 
tutorial) and is defined in <rtr_mac></rtr_mac> field.

Examples of this request adding L2 routes to interfaces
**veth1c** and **veth2c** are stored in 
[set_cont1_br_rt.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_cont1_br_rt.xml)
and
[set_cont2_br_rt.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_cont2_br_rt.xml)
files.

Values of **veth1c** and **veth2c** interfaces MAC address can be retrieved
using script **devmac2list** by running next commands from the host OS:

    sudo docker exec -ti cont1 bash /tut-rep/scripts/devmac2list veth1c
    sudo docker exec -ti cont1 bash /tut-rep/scripts/devmac2list veth2c

The output of each command must be substituted into **rtr_mac** field of
**set_cont1_br_rt.xml** and **set_cont2_br_rt.xml** inside **contrail-tools**
container (instead of lines `<element>1</element> ... <element>6</element>`).
Next we run **vrcli** command inside **contrail-tools**:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_cont1_br_rt.xml
    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_cont2_br_rt.xml

The added routes can be verified using **rt** command inside **contrail-tools**
container (here 0 corresponds to the label of the VRF table):

    rt --dump 0 --family bridge

The output of the command above must be similar to Fig. D-6. Although the
current configuration of vRouter Forwarder allows to trasmit L2 fragments
between containers **cont1** and **cont2**, it is of little use because
usually we operate with L3 packets and therefore we need L3 nexthops and
L3 routes in order to use ping. However, even this very simple configuration
demonstrates how data in general is organized inside vRouter Forwarder.
Namely, we have several interconnected tables responsible for transmission
of packets between virtual machines or containers (see Fig. D7):
- a VRFs table defining list of VRF tables with isolated routing information;
- an interfaces table specifying the association between an interface
(attached to a VM or a container) with a VRF table and with a nexthop;
- a nexthops table defining a list of nexthops (destinations);
- bridge and inet routes tables associated with a VRF table
defining which nexthops must be applied to packets depending on their
outer headers.

![Fig. D6: An example of "rt" output in contrail-tools with the list of new L2 routes](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-D-6.png)
*Fig. D6: An example of "rt" output in contrail-tools with the list of new L2 routes*


![Fig. D7: Relations between main OpenSDN vRouter Forwarder tables: the VRFs table, routes tables, the nexthop table, the interfaces table](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-D-7.jpg)
*Fig. D7: Relations between main OpenSDN vRouter Forwarder tables: the VRFs table, routes tables, the nexthop table, the interfaces table*. An arrow shows that an element references another one.

At the next step L3 routes are to be added to enable transmission of L3
packets through OpenSDN vRouter Forwarder. As with L2 routes, the corresponding
nexthops must be added beforehand. Examples of requests to add L3 nexthops
are stored in
[set_cont1_inet_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_cont1_inet_nh.xml)
and
[set_cont2_inet_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_cont2_inet_nh.xml)
files. 
Labels 11 and 22 were selected for L3 nexthops associated with interfaces
**veth1c** and **veth2c** (see Fig. D-7). The templates also require
modification of the interface MAC address in **nhr_encap** field (the procedure
is similar to the used previously for L2 nexthops) and the interface label
**nhr_encap_oif_id** must be set accordingly (same as for L2 nexthops). Finally,
the nexthops are added in **contrail-tools** container using the commands:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_cont1_inet_nh.xml
    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_cont2_inet_nh.xml

Upon the successful completion, the list of nexthops must get updated and this
can be verified using **nh** command inside **contrail-tools** container:

    nh --list

The reference output of the command is presented in Fig. D-8.

![Fig. D8: An example of "nh" output in contrail-tools with the list of new nexthops](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-D-8.png)
*Fig. D8: An example of "nh" output in contrail-tools with the list of new nexthops*

Finally, L3 routes are added using the requests similar to what have been
used for L2 routes. The examples of requests are stored in
[set_cont1_inet_rt.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_cont1_inet_rt.xml)
and
[set_cont2_inet_rt.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_cont2_inet_rt.xml)
files.
These route requests create new routes in IPv4 inet route table of VRF table 0
for prefixes 10.1.1.11/32 and 10.1.1.22/32 according to the initial network
configuration (see Fig. I-1). The route with prefix 10.1.1.11/32 points to
nexthop 11 and the route with prefix 10.1.1.22/32 points to
nexthop 22: this means that whenever a packet with the destination IP in the
outer header equal to 10.1.1.11/32 or 10.1.1.22/32 enters vRouter Forwarder
through interfaces associated with VRF table 0, it will be redirected
to the corresponding nexthops (11 and 22 respectively).

The routes are added in **contrail-tools** container using the commands:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_cont1_inet_rt.xml
    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_cont2_inet_rt.xml

The resulting configuration can be verified using **rt** command in
**contrail-tools** container:

    rt --get 10.1.1.11/32 --vrf 0 --family inet
    rt --get 10.1.1.22/32 --vrf 0 --family inet

The output of the commands should be similar to the presented on Fig. D-9.

![Fig. D9: An example of "rt" output in contrail-tools with the list of new L3 routes](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-D-9.png)
*Fig. D9: An example of "rt" output in contrail-tools with the list of new L3 routes*

E. Verification of connectivity between containers
--------------------------------------------------

It might look at this point that the current vRouter Forwarder configuration
is sufficient to transmit packages between containers **cont1** and **cont2**:
there are two interfaces connected to the containers and to vRouter Forwarder,
and there are L3 and L2 routes linked to these interfaces allowing transmission
of packets between them. Obviously, the configuration can be verified using
**ping** and **tcpdump** utilities:
1. start **tcpdump** in the host OS system watching interface 1 **veth1**:
        sudo tcpdump -i veth1 -vv -n
2. run **ping**  to the second container (**cont2**) inside the first
container **cont1**:
        ping -n 10.1.1.22

It is seen from the output (see Fig. E-1 for example) that there are no
ICMP requests from 10.1.1.11 to 10.1.1.22, therefore no responds from the
latter. However, there are many ARP requests
about resolution of MAC address for IP 10.1.1.11. Hence, it can be concluded
that possibly the current vRouter Forwarder configuration doesn't transmit
ARP packets from **cont1** to **cont2** or in the reverse direction. The
reasons of this behavour can be clarified using **dropstats** utility
from **contrail-tools** container:

    dropstats --log 0

This utility prints to the console recent error messages produced by vRouter
Forwarder during transmission of packets. Since error messages are stored
inside vRouter Forwarder separately for each CPU, the option **--log 0**
tells **dropstats** to collect data from all buffers into one output stream.
It is seen from the example output shown on Fig. E-2 that vRouter Forwarder
can't find an L2 router for broadcast MAC address FF:FF:FF:FF:FF:FF. 


![Fig. E1: An example of "tcpdump" with ARP requests](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-E-1.png)
*Fig. E1: An example of "tcpdump" with ARP requests*


![Fig. E2: An example of "dropstats" output in contrail-tools showind last forwarding error messages](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-E-2.png)
*Fig. E2: An example of "dropstats" output in contrail-tools showind last forwarding error messages*


Thus, we need a route with a corresponding nexthop that redirects ARP packets
destined to FF:FF:FF:FF:FF:FF to all virtual interfaces except the
ingress one. This type of nexthops are called Composite in OpenSDN, while
such routes are called multicast.

The examples of requests are stored in
[set_mcast_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_mcast_br_nh.xml)
and in
[set_mcast_br_rt.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_mcast_br_rt.xml).

The main difference with previous **vr_nexthop_req** can be observed in
[set_mcast_br_nh.xml](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/xml_reqs/set_mcast_br_nh.xml) because of two additional fields:
- <nhr_nh_list></nhr_nh_list>, a list of nexthops (sub-nexthops) associated with
virtual interfaces;
- <nhr_label_list></nhr_label_list>, a list of MPLS labels corresponding to
the nexthops from <nhr_nh_list></nhr_nh_list>.

Having applied the commands to run these requests from **contrail-tools**
container:

    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_mcast_br_nh.xml
    vrcli --vr_kmode --send_sandesh_req tut-rep/xml_reqs/set_mcast_br_rt.xml

we can now verify the modified list of nexthops and the
modified list of L2 routes (see Fig. E-3) have accommodated the new records.

![Fig. E3: An example of "nh" and "rt" output in contrail-tools showing new nexthops and bridge routes tables](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-E-3.png)
*Fig. E3: An example of "nh" and "rt" output in contrail-tools showing new nexthops and bridge routes tables*

Now **ping** command from 10.1.1.11 to 10.1.1.22 and in the reverse direction
must work properly (see Fig E-4).

Finally, it is recommended to check UDP connection using **nc** utility:

1) start an UDP server in **cont1**:

        nc -4 -u -l 10.1.1.11 50000

2) start an UDP client in **cont2**:

        nc -4 -u 10.1.1.11 50000

3) try typing messages in one **nc** window (**nc** prompt) to see
them appearing in another **nc** window.

![Fig. E4: An example of "ping" output in cont1 showing successfull ICMP connection between 10.1.1.11 and 10.1.1.22](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/blob/main/figs/Fig-E-4.png)
*Fig. E4: An example of "ping" output in cont1 showing successfull ICMP connection between 10.1.1.11 and 10.1.1.22*

Closure
-------

This section collects definitions of the main vRouter Forwarder
configuration elements employed for packets forwarding inside
OpenSDN dataplane and gives a brief overview of utilities for
managing and monitoring a virtual network configuration at this
layer.

If you find any mistake or inaccuraties in the text, please report them via
[Issues](https://github.com/mkraposhin/opensdn-forwarder-basic-tutorial/issues)
section of the repository.

### OpenSDN data plane basic defintions

Perhaps, this is the most important part of the document since it introduces
definitions which are mandatory for successfull administration, usage and
enhancement of OpenSDN virtual networks and applications.

So far we have encountered next data plane entities employed to organize 
communication between containers.

1. A **virtual interface** (or VIF) is a connected pair of interfaces with
one of them pointing into a virtual machine or a container and another one
attached to the host operating system where virtual resources (VMs or 
other type) are managed. Before being used with vRouter Forwarder a virtual
interface must be plugged in using the **vif** utility or a Sandesh
request.
2. A **VRF table** provides a unique and isolated environment to store
routing (L3) and forwarding (L2) tables for switching packets between
virtual interfaces associated with a given identifier (a number). The IP or MAC
addresses from two different VRF tables on a vRouter Forwarder can
intersect because of this isolation principle.
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

### OpenSDN dataplane utilities

Next OpenSDN tools (from **contrail-tools** image) can be used for manipulation and
inspection of the dataplane state:

- **vrcli** sends requests (specified in XML files) into vRouter Forwarder to
alter it's state;
- **vif** lists virtual and physcial interfaces attached to the vRouter
Forwarder;
- **vrftable** prints a list of VRF tables available on the vRouter Forwarder;
- **nh** gives a list of nexthops created at the vRouter Forwarder;
- **mpls** returns a list of available MPLS labels;
- **rt** prints all L3 and L2 routes from the vRouter Forwarder VRF tables;
- **dropstats** outputs error messages arising during packets transmission
in the vRouter Forwarder.

Bibliography
------------

1. RFC 4364: BGP/MPLS IP Virtual Private Networks (VPNs)
