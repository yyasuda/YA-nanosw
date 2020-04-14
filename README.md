# YA-nanosw
yet another simple L2 switch prototype

## to try this



### preparing 

```bash
== Compile
$ docker run -it -v /your/directory/of/YA-nanosw/:/tmp/ yutakayasuda/p4c_python3 /bin/bash
# p4c --target bmv2 --arch v1model --p4runtime-files p4info.txt nanosw05.p4 
#

== Start Mininet

$ docker run --privileged --rm -it -p 50001:50001 opennetworking/p4mn --arp --topo single,3 --mac

== Start P4Runtime Shell (for developer version)

$ docker run -it -v /your/directory/of/YA-nanosw/:/tmp/ yutakayasuda/p4runtime-shell-dev /bin/bash
root@f4c183e39939:/p4runtime-sh# source $VENV/bin/activate
(venv) root@f4c183e39939:/p4runtime-sh# 
(venv) root@f4c183e39939:/tmp# cp shell.py context.py /p4runtime-sh/p4runtime_sh/
(venv) root@f4c183e39939:/tmp# /p4runtime-sh/p4runtime-sh --grpc-addr 192.168.XX.XX:50001 --device-id 1 --election-id 0,1 --config /tmp/p4info.txt,nanosw05.json 
*** Welcome to the IPython shell for P4Runtime ***
P4Runtime sh >>>

== create Multicast Group

P4Runtime sh >>> me = MulticastGroupEntry(1).add(1).add(2).add(3)

P4Runtime sh >>> me.insert()

P4Runtime sh >>> 

```

[here](https://github.com/yyasuda/P4Runtime-firstbite/blob/master/t0_prepare.md) is helpful, as more detail explanation. [this one](https://github.com/yyasuda/P4Runtime-nanoswitch/blob/master/t1_nanosw01.md), maybe too.

### Experiment

1. Start PacketIn() on P4Runtime Shell
2. send a ping on Miniswitch
3. check the log and entries of tables

Actual example follows;

```bash
P4Runtime sh >>> PacketIn()

.....
======
packet-in: dst=00:00:00:00:00:02 src=00:00:00:00:00:01 port=1
## INSERT SRC ## src=00:00:00:00:00:01
## INSERT DST ## dst=00:00:00:00:00:01 port=1
.
======
packet-in: dst=00:00:00:00:00:01 src=00:00:00:00:00:02 port=2
## INSERT SRC ## src=00:00:00:00:00:02
## INSERT DST ## dst=00:00:00:00:00:02 port=2
...^C
Nothing (returned None)

P4Runtime sh >>> table_entry["MyIngress.l2_src_table"].read(lambda te: print(te))
table_id: 33565162 ("MyIngress.l2_src_table")
match {
  field_id: 1 ("hdr.ethernet.srcAddr")
  exact {
    value: "\\x00\\x00\\x00\\x00\\x00\\x01"
  }
}
action {
  action {
    action_id: 16784101 ("MyIngress.nop")
  }
}

table_id: 33565162 ("MyIngress.l2_src_table")
match {
  field_id: 1 ("hdr.ethernet.srcAddr")
  exact {
    value: "\\x00\\x00\\x00\\x00\\x00\\x02"
  }
}
action {
  action {
    action_id: 16784101 ("MyIngress.nop")
  }
}


P4Runtime sh >>> table_entry["MyIngress.l2_dst_table"].read(lambda te: print(te))
table_id: 33597842 ("MyIngress.l2_dst_table")
match {
  field_id: 1 ("hdr.ethernet.dstAddr")
  exact {
    value: "\\x00\\x00\\x00\\x00\\x00\\x01"
  }
}
action {
  action {
    action_id: 16838673 ("MyIngress.forward")
    params {
      param_id: 1 ("port")
      value: "\\x00\\x01"
    }
  }
}

table_id: 33597842 ("MyIngress.l2_dst_table")
match {
  field_id: 1 ("hdr.ethernet.dstAddr")
  exact {
    value: "\\x00\\x00\\x00\\x00\\x00\\x02"
  }
}
action {
  action {
    action_id: 16838673 ("MyIngress.forward")
    params {
      param_id: 1 ("port")
      value: "\\x00\\x02"
    }
  }
}


P4Runtime sh >>>
```

### behavior of packets

when you do 'ping h1 to h2';

1. the request packet comes to switch, the source MAC (h1) is not registered (unknown mac), send cpu as packet-In
2. controller registers h1 to both of src and dst table, and send packet back as packet-out
3. switch floods that packet
4. h2 recieved packet, and send back the reply packet to h1
5. the reply packet comes to switch, the source MAC (h2) is also unknown, send cpu again
6. controller registers h2 to src and dst table, then packet-out again
7. switch floods that packet too
8. the original host (h1) receive it

And once more, you do 'ping h1 to h2' (or h2 to h1), you will not see any packet-in/out. all packet forwardings will be done inside the switch according to registered entries.



