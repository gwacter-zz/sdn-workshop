# Mininet/Openflow 

#PreLab Setup

Note that this assumes you are using Linux and are using VirtualBox.

Open a terminal.

Start VirtualBox:

``
VirtualBox
``

Choose File -> Import Appliance

In the file dialog navigate to the directory /local/scratch/Mininet-VM

Choose the file Mininet-VM.ova

Log in with the username and password mininet

Find out what IP address is associated with your network interface to use for accessing the guest OS from the host OS. Note that we are using Ubuntu 16.04 so the new network interface naming conventions are the in place.

``
ifconfig enp0s8
``



ensureing that the VirtualBox network has been set to bridged adapter.
Devices->Network->Network setting-> bridged adapter

now you should be able to ping from the underlying host to the VM...


# Objectives

In this lab, you will start by learning the basics of running Mininet in a virtual machine. Mininet facilitates creating and manipulating Software Defined Networking components. Through mininet you will explore OpenFlow, which is an open interface for controlling the network elements through their forwarding tables. A network element may be converted into a switch, router or even an access points via low-level primitives defined in OpenFlow. This lab is your opportunity to gain hands-on experience with the platforms and debugging tools most useful for developing network control applications on OpenFlow.

* Access Mininet on your own private VM.
* Run the Ryu controller with a sample application
* Use various commands to gain experience with OpenFlow control of OpenvSwitch


# Network Topology

The topology we are using is similar to the one in the Openflow Tutorial (https://github.com/osrg/ryu/wiki/OpenFlow_Tutorial). It has three hosts named h1, h2 and h3 respectively. Each host has an Ethernet interface called h1-eth0, h2-eth0 and h3-eth0 respectively. The three hosts are connected through a switch names s1. The switch s1 has three ports named s1-eth1, s1-eth2 and s1-eth3. The controller is connected on the loopback interface (in real life this may or may not be the case, it means the switch and controller are built in a single box). The controller is identified as c0 and connected through port 6633.



	      +---------------------------+
	      |                           |
	      |      C0 - Controller      |
	      |                           |
	      +-------------+-------------+
	                    |
	      +-------------+-------------+
	      |                           |
	      |      S1 - OpenFlow        |
	      |          Switch           |
	      |                           |
	      +-+----------+----------+---+
	      s1-eth0    s1-eth1    s1-eth2
	       +            +            +
	       |            |            |
	       |            |            |
	       v            v            v
	 h1-eth0         h2-eth0         h3-eth0
	 +-+--+          +-+--+          +-+--+
	 | H1 |          | H2 |          | H3 |
	 +----+          +----+          +----+


At the command prompt type,

> mn --topo single,3 --mac --controller remote --switch ovsk
 
This should present you with the following terminal output..
 
mininet>

You are now in a virtual network you have created, we can do interesting things in this network between the nodes!!!


