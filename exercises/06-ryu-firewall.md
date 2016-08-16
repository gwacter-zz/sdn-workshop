# Exercise 6

# Ryu Firewall

See: https://osrg.github.io/ryu-book/en/html/rest_firewall.html

This exercise doesn't require X installed on the Mininet VM.

# Starting the RYU Openflow controller

## Ensure that no other controller is present

```
root@mininet-vm:~# killall controller
controller: no process found
root@mininet-vm:~#
```

Note that 'controller' is a simple OpenFlow reference controller implementation in linux.  
We want to ensure that this is not running before we start our own controller.

## Clear all mininet components

```
root@mininet-vm:~# mn -c

```

## Start the Ryu Firewall application

For convenience set an environment variable that points at the directory containing the Ryu packages. 

The example below is for my system, yours should be similar.

```
$ export RYU_APP=/usr/local/lib/python2.7/dist-packages/ryu/app/
```

To execute a Ryu application you need to use the ``ryu-manager``. 

```
$ ryu-manager --verbose $RYU_APP/ryu.app.rest_firewall
```

You should see:
```
loading app /usr/local/lib/python2.7/dist-packages/ryu/app//ryu.app.rest_firewall
loading app ryu.controller.ofp_handler
instantiating app None of DPSet
creating context dpset
creating context wsgi
instantiating app /usr/local/lib/python2.7/dist-packages/ryu/app//ryu.app.rest_firewall of RestFirewallAPI
instantiating app ryu.controller.ofp_handler of OFPHandler
BRICK dpset
  PROVIDES EventDP TO {'RestFirewallAPI': set(['dpset'])}
  CONSUMES EventOFPStateChange
  CONSUMES EventOFPPortStatus
  CONSUMES EventOFPSwitchFeatures
BRICK RestFirewallAPI
  CONSUMES EventDP
  CONSUMES EventOFPStatsReply
  CONSUMES EventOFPPacketIn
  CONSUMES EventOFPFlowStatsReply
BRICK ofp_event
  PROVIDES EventOFPStatsReply TO {'RestFirewallAPI': set(['main'])}
  PROVIDES EventOFPFlowStatsReply TO {'RestFirewallAPI': set(['main'])}
  PROVIDES EventOFPPacketIn TO {'RestFirewallAPI': set(['main'])}
  PROVIDES EventOFPStateChange TO {'dpset': set(['main', 'dead'])}
  PROVIDES EventOFPPortStatus TO {'dpset': set(['main'])}
  PROVIDES EventOFPSwitchFeatures TO {'dpset': set(['config'])}
  CONSUMES EventOFPPortDescStatsReply
  CONSUMES EventOFPHello
  CONSUMES EventOFPErrorMsg
  CONSUMES EventOFPEchoRequest
  CONSUMES EventOFPEchoReply
  CONSUMES EventOFPSwitchFeatures
(18761) wsgi starting up on http://0.0.0.0:8080
```

The controller is now running but not doing anything because no switches are connected to it.

# Using the Firewall

## Start mininet with 3 hosts connected to 1 switch

We are going to use the network architecture shown below.

<img src="https://osrg.github.io/ryu-book/en/html/_images/fig13.png" alt="example network, one switch and three hosts" title="example network" />()

First, build an environment on Mininet. 

```
$ sudo mn --topo single,3 --mac --switch ovsk --controller remote -x
*** Creating network
*** Adding controller
Unable to contact the remote controller at 127.0.0.1:6633
*** Adding hosts:
h1 h2 h3
*** Adding switches:
s1
*** Adding links:
(h1, s1) (h2, s1) (h3, s1)
*** Configuring hosts
h1 h2 h3
*** Running terms on localhost:10.0
*** Starting controller
*** Starting 1 switches
s1

*** Starting CLI:
mininet>```
```

##Ensure that the bridge is using OpenFlow13

```
mininet> sh ovs-vsctl set bridge s1 protocols=OpenFlow13
```

##Monitor controller to ensure that the switch connects

In the RYU controller window you should see a message similar to the following to show that the switch has connected to the controller and has exchanged information about its capabilities.

```
connected socket:<eventlet.greenio.base.GreenSocket object at 0x7f92b3e01150> address:('127.0.0.1', 38638)
EVENT ofp_event->dpset EventOFPStateChange
connected socket:<eventlet.greenio.base.GreenSocket object at 0x7f92b3e01b50> address:('127.0.0.1', 38640)
hello ev <ryu.controller.ofp_event.EventOFPHello object at 0x7f92b3e01750>
move onto config mode
EVENT ofp_event->dpset EventOFPSwitchFeatures
switch features ev version=0x4,msg_type=0x6,msg_len=0x20,xid=0x66e6e688,OFPSwitchFeatures(auxiliary_id=0,capabilities=79,datapath_id=1,n_buffers=256,n_tables=254)
move onto main mode
EVENT ofp_event->dpset EventOFPStateChange
EVENT ofp_event->dpset EventOFPPortStatus
DPSET: register datapath <ryu.controller.controller.Datapath object at 0x7f92b3e016d0>
EVENT dpset->RestFirewallAPI EventDP
DPSET: A port was modified.(datapath id = 0000000000000001, port number = 4294967294)
[FW][INFO] dpid=0000000000000001: Join as firewall.
```

## Dump flows on switch s1

A flow is the most fine-grained work unit of a switch. In Mininet, dpctl is a command that allows visibility and control over a single switch's flow table. It is especially useful for debugging, by viewing flow state and flow counters.

```
$ dpctl dump-flows -O OpenFlow13
*** s1 ------------------------------------------------------------------------
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=484.137s, table=0, n_packets=39, n_bytes=2430, priority=65535 actions=drop
 cookie=0x0, duration=484.137s, table=0, n_packets=0, n_bytes=0, priority=0 actions=CONTROLLER:128
 cookie=0x0, duration=484.137s, table=0, n_packets=0, n_bytes=0, priority=65534,arp actions=NORMAL
```

The flow syntax is described in the man page for dpct (http://ranosgrant.cocolog-nifty.com/openflow/dpctl.8.html) under FLOW SYNTAX.

Note that the output omits fields that are set to matched against ANY. Only specific matches are included.

The fields 'n_packets' and 'n_bytes' tell us how many packets and the volume of packets that matched the rules for a given flow entry.

There are three flows installed here are (in order shown above):

   * any packet, action is drop packet (priority=65535) ... 
   * any packet, action is send the first 128 bytes of the packet as a PACKET_IN message to controller (priority 0)
   * any ARP packet, action is to process as you would in a NORMAL switch (priority 65534)

Rules are matched in terms of the most specific and priorities are used to break ties (the highest number wins).

You see from above that the switch behaviour specified by the flows is to process ARP packets normally and drop all other packets.






# Passing packets

##Start a ping from host h1 to host h2
Mininet Window

```
mininet> h1 ping h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_req=1 ttl=64 time=5.10 ms
64 bytes from 10.0.0.2: icmp_req=2 ttl=64 time=0.238 ms
64 bytes from 10.0.0.2: icmp_req=3 ttl=64 time=0.052 ms
64 bytes from 10.0.0.2: icmp_req=4 ttl=64 time=0.051 ms
^C
--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
rtt min/avg/max/mdev = 0.051/1.360/5.100/2.160 ms
mininet>
```

## Monitor new messages in the controller window

In the RYU controller window we want to ensure that we see the EventOFPPacketIn messages along with the controller telling us that it is adding unicast flows.

```
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:01 ff:ff:ff:ff:ff:ff 1
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:02 00:00:00:00:00:01 2
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:01 00:00:00:00:00:02 1
```

# Dump flows again to view differences.

We can confirm that the unicast flows have been added by dumping the flow table on the switch

```
mininet>  dpctl dump-flows -O OpenFlow13
*** s1 ------------------------------------------------------------------------
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=112.044s, table=0, n_packets=3, n_bytes=182, priority=0 actions=CONTROLLER:65535
 cookie=0x0, duration=40.563s, table=0, n_packets=3, n_bytes=238, priority=1,in_port=2,dl_dst=00:00:00:00:00:01 actions=output:1
 cookie=0x0, duration=40.559s, table=0, n_packets=2, n_bytes=140, priority=1,in_port=1,dl_dst=00:00:00:00:00:02 actions=output:2
mininet>
```

# Running with more hosts.

## Stop the current Mininet simulation

```
mininet> exit
*** Stopping 1 switches
s1 ...
*** Stopping 3 hosts
h1 h2 h3
*** Stopping 1 controllers
c0
*** Done
completed in 3.678 seconds
root@mininet-vm:~#
```

## Start a new simulation with a few more hosts (10 hosts, 1 switch)

```
root@mininet-vm:~# mn --topo=tree,1,10 --mac --controller=remote --switch ovsk,protocols=OpenFlow13
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 h3 h4 h5 h6 h7 h8 h9 h10
*** Adding switches:
s1
*** Adding links:
(h1, s1) (h2, s1) (h3, s1) (h4, s1) (h5, s1) (h6, s1) (h7, s1) (h8, s1) (h9, s1) (h10, s1)
*** Configuring hosts
h1 h2 h3 h4 h5 h6 h7 h8 h9 h10
*** Starting controller
*** Starting 1 switches
s1
*** Starting CLI:
mininet>
```

##Ensure that the bridge is using OpenFlow13

```
mininet> sh ovs-vsctl set bridge s1 protocols=OpenFlow13
```

## Dump flows again to view differences.

```
mininet> dpctl dump-flows -O OpenFlow13
*** s1 ------------------------------------------------------------------------
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=6.084s, table=0, n_packets=0, n_bytes=0, priority=0 actions=CONTROLLER:65535
mininet>
```

## Ping from h1 to h2 once again

Mininet Window

```
mininet> h1 ping h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_req=1 ttl=64 time=0.585 ms
64 bytes from 10.0.0.2: icmp_req=2 ttl=64 time=0.319 ms
64 bytes from 10.0.0.2: icmp_req=3 ttl=64 time=0.063 ms
^C
--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.063/0.322/0.585/0.213 ms
mininet>
```

## Dump flows again to view differences.
Rootshell Window


```
mininet> dpctl dump-flows -O OpenFlow13
*** s1 ------------------------------------------------------------------------
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=36.095s, table=0, n_packets=3, n_bytes=182, priority=0 actions=CONTROLLER:65535
 cookie=0x0, duration=7.464s, table=0, n_packets=6, n_bytes=532,priority=1,in_port=2,dl_dst=00:00:00:00:00:01 actions=output:1
 cookie=0x0, duration=7.462s, table=0, n_packets=5, n_bytes=434,priority=1,in_port=1,dl_dst=00:00:00:00:00:02 actions=output:2
mininet>
```

## Ping all hosts

Mininet Window

```
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 h3 h4 h5 h6 h7 h8 h9 h10
h2 -> h1 h3 h4 h5 h6 h7 h8 h9 h10
h3 -> h1 h2 h4 h5 h6 h7 h8 h9 h10
h4 -> h1 h2 h3 h5 h6 h7 h8 h9 h10
h5 -> h1 h2 h3 h4 h6 h7 h8 h9 h10
h6 -> h1 h2 h3 h4 h5 h7 h8 h9 h10
h7 -> h1 h2 h3 h4 h5 h6 h8 h9 h10
h8 -> h1 h2 h3 h4 h5 h6 h7 h9 h10
h9 -> h1 h2 h3 h4 h5 h6 h7 h8 h10
h10 -> h1 h2 h3 h4 h5 h6 h7 h8 h9
*** Results: 0% dropped (90/90 received)
mininet>
```

## Monitor new messages in the controller window
RYU Window
```
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:07 ff:ff:ff:ff:ff:ff 7
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:09 00:00:00:00:00:07 9
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:07 00:00:00:00:00:09 7
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:07 ff:ff:ff:ff:ff:ff 7
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:0a 00:00:00:00:00:07 10
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:07 00:00:00:00:00:0a 7
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:08 ff:ff:ff:ff:ff:ff 8
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:09 00:00:00:00:00:08 9
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:08 00:00:00:00:00:09 8
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:08 ff:ff:ff:ff:ff:ff 8
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:0a 00:00:00:00:00:08 10
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:08 00:00:00:00:00:0a 8
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:09 ff:ff:ff:ff:ff:ff 9
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:0a 00:00:00:00:00:09 10
EVENT ofp_event->SimpleSwitch13 EventOFPPacketIn
packet in 1 00:00:00:00:00:09 00:00:00:00:00:0a 9
```


## Dump flows again to view differences.
```
mininet> dpctl dump-flows -O OpenFlow13
*** s1 ------------------------------------------------------------------------
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=100.69s, table=0, n_packets=135, n_bytes=8190, priority=0 actions=CONTROLLER:65535
 cookie=0x0, duration=41.876s, table=0, n_packets=2, n_bytes=140, priority=1,in_port=1,dl_dst=00:00:00:00:00:07 actions=output:7
 cookie=0x0, duration=41.624s, table=0, n_packets=3, n_bytes=238, priority=1,in_port=9,dl_dst=00:00:00:00:00:08 actions=output:8
 cookie=0x0, duration=41.692s, table=0, n_packets=3, n_bytes=238, priority=1,in_port=10,dl_dst=00:00:00:00:00:05 actions=output:5
 cookie=0x0, duration=41.765s, table=0, n_packets=2, n_bytes=140, priority=1,in_port=3,dl_dst=00:00:00:00:00:0a actions=output:10
 cookie=0x0, duration=41.622s, table=0, n_packets=2, n_bytes=140, priority=1,in_port=8,dl_dst=00:00:00:00:00:09 actions=output:9
 cookie=0x0, duration=41.882s, table=0, n_packets=2, n_bytes=140, priority=1,in_port=1,dl_dst=00:00:00:00:00:06 actions=output:6
 cookie=0x0, duration=41.738s, table=0, n_packets=2, n_bytes=140, priority=1,in_port=4,dl_dst=00:00:00:00:00:08 actions=output:8
 cookie=0x0, duration=41.821s, table=0, n_packets=2, n_bytes=140, priority=1,in_port=2,dl_dst=00:00:00:00:00:08 actions=output:8
 cookie=0x0, duration=41.783s, table=0, n_packets=2, n_bytes=140, priority=1,in_port=3,dl_dst=00:00:00:00:00:07 actions=output:7
 ...
 ...
 ...
```

## Ping all hosts once again

Mininet Window

```
mininet> pingall
*** Ping: testing ping reachability
h1 -> h2 h3 h4 h5 h6 h7 h8 h9 h10
h2 -> h1 h3 h4 h5 h6 h7 h8 h9 h10
h3 -> h1 h2 h4 h5 h6 h7 h8 h9 h10
h4 -> h1 h2 h3 h5 h6 h7 h8 h9 h10
h5 -> h1 h2 h3 h4 h6 h7 h8 h9 h10
h6 -> h1 h2 h3 h4 h5 h7 h8 h9 h10
h7 -> h1 h2 h3 h4 h5 h6 h8 h9 h10
h8 -> h1 h2 h3 h4 h5 h6 h7 h9 h10
h9 -> h1 h2 h3 h4 h5 h6 h7 h8 h10
h10 -> h1 h2 h3 h4 h5 h6 h7 h8 h9
*** Results: 0% dropped (90/90 received)
mininet>
```

## Monitor new messages in the controller window
RYU Window

What happened that time?


# Running a high bandwidth flow

## Starting iperf between hosts

```
mininet> iperf
*** Iperf: testing TCP bandwidth between h1 and h10
waiting for iperf to start up...*** Results: ['22.5 Gbits/sec', '22.6 Gbits/sec']
mininet>
```

## Dump flows to see the flows which match

```
mininet> dpctl dump-flows -O OpenFlow13
*** s1 ------------------------------------------------------------------------
OFPST_FLOW reply (OF1.3) (xid=0x2):
cookie=0x0, duration=188.223s, table=0, n_packets=135, n_bytes=8190, priority=0 actions=CONTROLLER:65535
...
...
cookie=0x0, duration=129.393s, table=0, n_packets=207403, n_bytes=13688658, priority=1,in_port=10,dl_dst=00:00:00:00:00:01 actions=output:1
...
...
cookie=0x0, duration=129.391s, table=0, n_packets=290931, n_bytes=14129626606, priority=1,in_port=1,dl_dst=00:00:00:00:00:0a actions=output:10
...
...
```

Did any packets come to the controller?

Where were most of the packets sent?

# More Information

Checkout: http://ryu.readthedocs.io/en/latest/index.html

In particular writing your first application: http://ryu.readthedocs.io/en/latest/writing_ryu_app.html
