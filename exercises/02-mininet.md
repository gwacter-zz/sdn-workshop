# Excercise 1

##Try these useful mininet commands
> mininet command help
```
mininet> help
```
 
> Topology information
```
mininet> net
```
 
> IP address information
```
mininet> dump
```
 
> List of nodes and switches
```
mininet> nodes
```
 
> Ping from each host to all other hosts
```
mininet> pingall
```
 
##Try these ovs-ofctl commands. (Use sh to execute a shell command from in mininet.)
> ovs-ofctl command help
```
mininet> sh ovs-ofctl --help | less
```
 
> Shows switch information including OpenFlow Feature support, port type, port speed and mac addresses.
```
mininet> sh ovs-ofctl show s1
```
 
> Shows active flow rules on the switch
```
mininet> sh ovs-ofctl dump-flows s1
```
 
> Show port traffic and error statistics
```
mininet> sh ovs-ofctl dump-ports s1
```
 
## Add Flows to the switch
```
mininet> sh ovs-ofctl add-flow s1 arp,idle_timeout=180,priority=500,actions=output:1,2,3
mininet> sh ovs-ofctl add-flow s1 tcp,tp_dst=80,nw_dst=10.0.0.3,actions=output:3
mininet> sh ovs-ofctl add-flow s1 ip,nw_dst=10.0.0.1,actions=output:1
```
`
## Status Check
```
mininet> sh ovs-ofctl dump-flows s1
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=47.91s, table=0, n_packets=0, n_bytes=0, idle_age=47, ip,nw_dst=10.0.0.1 actions=output:1
 cookie=0x0, duration=73.85s, table=0, n_packets=18, n_bytes=756, idle_timeout=180, idle_age=1, priority=500,arp actions=output:1,output:2,output:3
 cookie=0x0, duration=62.666s, table=0, n_packets=0, n_bytes=0, idle_age=62, tcp,nw_dst=10.0.0.3,tp_dst=80 actions=output:3
```
## Start a web server on h3
```
mininet> h3 python -m SimpleHTTPServer 80 &
```
## Retrieve the web page from h1
```
mininet> h1 wget http://10.0.0.3
--2015-10-12 02:00:26--  http://10.0.0.3/
Connecting to 10.0.0.3:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 530 [text/html]
Saving to: ‘index.html.1’

     0K                                                       100% 49.5M=0s

2015-10-12 02:00:26 (49.5 MB/s) - ‘index.html.1’ saved [530/530]
```

## Can you ping?
```
mininet> pingall
```
## Why not?
## Why might you use the following flow rules instead?
```
mininet> sh ovs-ofctl add-flow s1 arp,in_port=1,actions=output:3
mininet> sh ovs-ofctl add-flow s1 arp,in_port=3,actions=output:1
```
