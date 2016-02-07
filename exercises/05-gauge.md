# Adding a second application (Gauge)

It is possible to add a second application to our data path.  This application won't be used to alter packet flow, but instead it will be used to monitor port statistics. Open a third ssh window and change to root.

```
$ssh sysadm@10.10.0.<NUMBER> 
$sudo bash
# cd faucet
# cp gauge.conf-dist /etc/opt/faucet/gauge.conf
```

## Add a second controller inside mininet

We need to tell the mininet application that there are now two controllers.

```
mininet> sh ovs-vsctl set-controller s1 tcp:127.0.0.1:6633 tcp:127.0.0.1:6653
```

## Start the Gauge application

In the third window, we start the Gauge application on a differnet port to stop collisions

```
# ryu-manager --verbose  --ofp-tcp-listen-port=6653 gauge.py
```

## Monitor port statistics

If you open a new window, you can monitor the statistics file and see new port stats written to the file every 10 seconds.

```
ssh sysadm@10.10.0.<NUMBER>
$tail -f /var/log/faucet/ports.out
```

```
Oct 11 01:50:52	test-faucet-1-port2-packets-out	87
Oct 11 01:50:52	test-faucet-1-port2-packets-in	38
Oct 11 01:50:52	test-faucet-1-port2-bytes-out	4326
Oct 11 01:50:52	test-faucet-1-port2-bytes-in	2308
Oct 11 01:50:52	test-faucet-1-port2-dropped-out	0
Oct 11 01:50:52	test-faucet-1-port2-dropped-in	0
Oct 11 01:50:52	test-faucet-1-port2-errors-in	0
```
