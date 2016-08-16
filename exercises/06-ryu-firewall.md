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
mininet> dpctl dump-flows -O OpenFlow13
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

This reflects that immediately after starting the firewall, it was set to disable status to cut off all communication. 

Enable it with the following command.

```
$ curl -X PUT http://localhost:8080/firewall/module/enable/0000000000000001
  [
    {
      "switch_id": "0000000000000001",
      "command_result": {
        "result": "success",
        "details": "firewall running."
      }
    }
  ]
```

Check that is enabled.

```
$ curl http://localhost:8080/firewall/module/status
  [
    {
      "status": "enable",
      "switch_id": "0000000000000001"
    }
  ]
```

Now dump the flow table to see what has changed:

```
mininet> dpctl dump-flows -O OpenFlow13
*** s1 ------------------------------------------------------------------------
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=3158.793s, table=0, n_packets=0, n_bytes=0, priority=65534,arp actions=NORMAL
 cookie=0x0, duration=3158.793s, table=0, n_packets=0, n_bytes=0, priority=0 actions=CONTROLLER:128
```

See that the DROP flow has been removed.

Now the rules say that ARP is handled normally but anything else is sent to the controller.

Trying to ping will result in the packet_n for the second rule increasing and you will see the controller log register that it 
is receiving packets.

The Firewall controller will block packets unless explicitly allowed.

## Add a Rule

Add a rule to permit pinging between h1 and h2. You need to add the rule for both ways.

Let’s add the following rules. Rule ID is assigned automatically.

| Source |	Destination |	Protocol | 	Permission | 	(Rule ID) |
| 10.0.0.1/32 |	10.0.0.2/32 |	ICMP |	Allow |	1 |
| 10.0.0.2/32 |	10.0.0.1/32 |	ICMP | 	Allow | 2 |

Use the RESTful interface:

```
$ curl -X POST -d '{"nw_src": "10.0.0.1/32", "nw_dst": "10.0.0.2/32", "nw_proto": "ICMP"}' http://localhost:8080/firewall/rules/0000000000000001
  [
    {
      "switch_id": "0000000000000001",
      "command_result": [
        {
          "result": "success",
          "details": "Rule added. : rule_id=1"
        }
      ]
    }
  ]

$ curl -X POST -d '{"nw_src": "10.0.0.2/32", "nw_dst": "10.0.0.1/32", "nw_proto": "ICMP"}' http://localhost:8080/firewall/rules/0000000000000001
  [
    {
      "switch_id": "0000000000000001",
      "command_result": [
        {
          "result": "success",
          "details": "Rule added. : rule_id=2"
        }
      ]
    }
  ]
```

Added rules are registered in the switch as flow entries.

```
mininet> dpctl dump-flows -O OpenFlow13
*** s1 ------------------------------------------------------------------------
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=3636.260s, table=0, n_packets=4, n_bytes=168, priority=65534,arp actions=NORMAL
 cookie=0x1, duration=26.389s, table=0, n_packets=0, n_bytes=0, priority=1,icmp,nw_src=10.0.0.1,nw_dst=10.0.0.2 actions=NORMAL
 cookie=0x2, duration=8.975s, table=0, n_packets=0, n_bytes=0, priority=1,icmp,nw_src=10.0.0.2,nw_dst=10.0.0.1 actions=NORMAL
 cookie=0x0, duration=3636.260s, table=0, n_packets=15, n_bytes=1470, priority=0 actions=CONTROLLER:128
```

You can see that ICMP is now allowed and passed to the switch pipeline for normal processing (note here we aren't going 100% openflow because instead of requiring the controller to work out ports etc. as per the simple switch, here we rely on the presence of a traditional pipeline).

## Deleting rules

Deleting a Rule

Delete the “rule_id:1” and “rule_id:2” rules.

Node: c0 (root):

```
$ curl -X DELETE -d '{"rule_id": "1"}' http://localhost:8080/firewall/rules/0000000000000001
  [
    {
      "switch_id": "0000000000000001",
      "command_result": [
        {
          "result": "success",
          "details": "Rule deleted. : ruleID=1"
        }
      ]
    }
  ]

$ curl -X DELETE -d '{"rule_id": "2"}' http://localhost:8080/firewall/rules/0000000000000001
  [
    {
      "switch_id": "0000000000000001",
      "command_result": [
        {
          "result": "success",
          "details": "Rule deleted. : ruleID=2"
        }
      ]
    }
  ]
```

You can see now the flow table has now been returned to it state prior to allowing the traffic.

```
mininet> dpctl dump-flows -O OpenFlow13
*** s1 ------------------------------------------------------------------------
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=4205s, table=0, n_packets=6, n_bytes=252, priority=65534,arp actions=NORMAL
 cookie=0x0, duration=4205s, table=0, n_packets=17, n_bytes=1666, priority=0 actions=CONTROLLER:128
mininet> 
```

## Examining the Code

The code is found here:
https://github.com/osrg/ryu/blob/master/ryu/app/rest_firewall.py

Useful resouces:
   * http://ryu.readthedocs.io/en/latest/getting_started.html -- the documentation
   * https://osrg.github.io/ryu-book/en/html/rest_api.html -- explains how to incorporate a REST interface into a controller

Ryu applications are single-threaded entities which implement various functionalities in Ryu. Events are messages between them.

Ryu applications send asynchronous events each other. Besides that, there are some Ryu-internal event sources which are not Ryu applications. One of examples of such event sources is OpenFlow controller. While an event can currently contain arbitrary python objects, it’s discouraged to pass complex objects (eg. unpickleable objects) between Ryu applications.

Each Ryu application has a receive queue for events. The queue is FIFO and preserves the order of events. Each Ryu application has a thread for event processing. The thread keep draining the receive queue by dequeueing an event and calling the appropritate event handler for the event type. Because the event handler is called in the context of the event processing thread, it should be careful for blocking. I.e. while an event handler is blocked, no further events for the Ryu application will be processed.

There are kinds of events which are used to implement synchronous inter-application calls between Ryu applications. While such requests uses the same machinary as ordinary events, their replies are put on another queue dedicated to replies to avoid deadlock. (Because, unlike erlang, our queue doesn’t support selective receive.) It’s assumed that the number of in-flight synchronous requests from a Ryu application is at most 1.

While threads and queues is currently implemented with eventlet/greenlet, a direct use of them in a Ryu application is strongly discouraged.

## Firewall

The Firewall makes use of the Ryu Web server function corresponding to WSGI. By using this function, it is possible to create a REST API, which is useful to link with other systems or browsers. Note that WSGI is a unified framework for connecting Web applications and Web servers in Python.

The first class isRestFirewallAPI(app_manager.RyuApp) which is the first class instantiated by the ryu framework (see _init) and initialises the rest of the application including defining the mapping between HTTP requests and local code to execute.

It also handles the following events generated by Ryu:
   * switch being connection and disconnection (EventDP), delegates handling these to FirewallController instance
   * openflow message ofpstatsreply and delegates this to the superclass RyuApp
   * openflow packetin message (i.e. we got handed a packet from the switch) and delegates this to FirewallController's packet_in_handler

The second class is FirewallOfsList(dict) which is a dictionary of switches (instances of ryu.controller.controller.Datapath), this supports one method get_ofs which returns an instance of a datapath given its id (represented as a string).

The third class is FirewallController(ControllerBase) which defines the methods invoked by the REST API at runtime as well as exposes methods for handling switch connection and disconnection as well as handling packets arriving at the controller. The switch connection/disconnection will create/destroy instances of Firewall. The REST API will maintain the switch flow table such that it expresses the current firewall rules. The packet handler will also receiver a packet, log what it is to the console and drop it.

The fourth class is Firewall(object) which contains the functionality related to writing flows to the switch (see http://ryu.readthedocs.io/en/latest/ofproto_v1_3_ref.html#modify-state-messages) as trigerred by calls to the RESTful interface as well as maintaining an abstraction representing the firewall's state. 

The fifth and sixth classes are utility classes for handling matching and action processing.
The second class is SimpleSwitchRest13, which is extension of ” Switching Hub ”, to be able to update the MAC address table.

With SimpleSwitchRest13, because flow entry is added to the switch, the FeaturesReply method is overridden and holds the datapath object.

## Changing the Firewall

To add functionality to mass load rules, you could add another restful interface to trigger it and implement the logic in FirewallController and Firewall.

Alternatively you could use signals (https://docs.python.org/2/library/signal.html).

To implement automatic bidirectional TCP traffic, you would need to extend how packet_in packers are managed.

