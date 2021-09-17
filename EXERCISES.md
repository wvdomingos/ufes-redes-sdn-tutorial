# Exercise 1 - 3: Learning the Basics


* [Exercise 1: P4Runtime Basics](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-1.md)
* [Exercise 2: Yang, OpenConfig, and gNMI Basics](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-2.md)
* [Exercise 3: Using ONOS as the control plane for Stratum](https://github.com/opennetworkinglab/ngsdn-tutorial/blob/advanced/EXERCISE-3.md)

### Topologia
<img src='img\topo-v6.png'>

## P4Runtime Basics
### Compile P4 program
```
make p4-build
```

### Start Mininet topology
```
make start
```

### Show the Mininet log
```
make mn-log
```

### Program leaf1 using P4Runtime
```
util/p4rt-sh --grpc-addr localhost:50001 --config p4src/build/p4info.txt,p4src/build/bmv2.json --election-id 0,1
```

### Attach to the Mininet CLI
```
make mn-cli
```

### Insert static NDP entries
```
mininet> h1a ip -6 neigh replace 2001:1:1::B lladdr 00:00:00:00:00:1B dev h1a-eth0
mininet> h1a ip -6 neigh replace 2001:1:1::C lladdr 00:00:00:00:00:1C dev h1a-eth0

mininet> h1b ip -6 neigh replace 2001:1:1::A lladdr 00:00:00:00:00:1A dev h1b-eth0
mininet> h1b ip -6 neigh replace 2001:1:1::C lladdr 00:00:00:00:00:1C dev h1b-eth0

mininet> h1c ip -6 neigh replace 2001:1:1::A lladdr 00:00:00:00:00:1A dev h1c-eth0
mininet> h1c ip -6 neigh replace 2001:1:1::B lladdr 00:00:00:00:00:1B dev h1c-eth0

mininet> h1a ping h1b
```

### Insert P4Runtime table entries
```
P4Runtime sh >>> te = table_entry['IngressPipeImpl.l2_exact_table'](action='IngressPipeImpl.set_egress_port')
                 te.match['hdr.ethernet.dst_addr'] = '00:00:00:00:00:1A'
                 te.action['port_num'] = '3'
                 te.insert()

P4Runtime sh >>> te = table_entry['IngressPipeImpl.l2_exact_table'](action='IngressPipeImpl.set_egress_port')
                 te.match['hdr.ethernet.dst_addr'] = '00:00:00:00:00:1B'
                 te.action['port_num'] = '4'
                 te.insert()

P4Runtime sh >>> te = table_entry['IngressPipeImpl.l2_exact_table'](action='IngressPipeImpl.set_egress_port')
                 te.match['hdr.ethernet.dst_addr'] = '00:00:00:00:00:1C'
                 te.action['port_num'] = '5'
                 te.insert()

P4Runtime sh >>> print(te)

P4Runtime sh >>> for te in table_entry["IngressPipeImpl.l2_exact_table"].read():
    			print(te)
```

## Yang, OpenConfig, and gNMI Basics
### Understanding the YANG language
We start with a simple YANG module called demo-port in yang/demo-port.yang

Start by entering the yang-tools Docker container:
```
$ make yang-tools
bash-4.4#
```

Next, run pyang on the demo-port.yang model:
```
bash-4.4# pyang -f tree demo-port.yang
```

We can also use pyang to visualize a more complicated set of models, like the set of OpenConfig models that Stratum uses.
These models have already been loaded into the yang-tools container in the /models directory.
```
bash-4.4# pyang -f tree \
    -p ietf \
    -p openconfig \
    -p hercules \
    openconfig/interfaces/openconfig-interfaces.yang \
    openconfig/interfaces/openconfig-if-ethernet.yang  \
    openconfig/platform/* \
    openconfig/qos/* \
    openconfig/system/openconfig-system.yang \
    hercules/openconfig-hercules-*.yang  | less
```

### Understand YANG encoding

There is no specific YANG data encoding, but data adhering to YANG models can be encoded into XML, JSON, or Protobuf (among other formats). Each of these formats has it's own schema format.
First, we can look at YANG's first and canonical representation format XML. To see a empty skeleton of data encoded in XML, run:
```
bash-4.4# pyang -f sample-xml-skeleton demo-port.yang
```

We can also use pyang to generate a DSDL schema based on the YANG model:
```
bash-4.4# pyang -f dsdl demo-port.yang | xmllint --format -
```

Next, we will look at encoding data using Protocol Buffers (protobuf). The protobuf encoding is a more compact binary encoding than XML, and libraries can be automatically generated for dozens of languages. We can use ygot's proto_generator to generate protobuf messages from our YANG model.
```
bash-4.4# proto_generator -output_dir=/proto -package_name=tutorial demo-port.yang
```

proto_generator will generate two files:
```
bash-4.4# less /proto/tutorial/demo_port/demo_port.proto
bash-4.4# less /proto/tutorial/enums/enums.proto
```

We can also use proto_generator to build the protobuf messages for the OpenConfig models that Stratum uses:
```
bash-4.4# proto_generator \    
-generate_fakeroot \    
-output_dir=/proto \    
-package_name=openconfig \   
-exclude_modules=ietf-interfaces \    
-compress_paths \    
-base_import_path= \    
-path=ietf,openconfig,hercules \    
openconfig/interfaces/openconfig-interfaces.yang \    
openconfig/interfaces/openconfig-if-ip.yang \    
openconfig/lacp/openconfig-lacp.yang \    
openconfig/platform/openconfig-platform-linecard.yang \    
openconfig/platform/openconfig-platform-port.yang \    
openconfig/platform/openconfig-platform-transceiver.yang \    
openconfig/platform/openconfig-platform.yang \   
openconfig/system/openconfig-system.yang \    
openconfig/vlan/openconfig-vlan.yang \    
hercules/openconfig-hercules-interfaces.yang \    
hercules/openconfig-hercules-platform-chassis.yang \    
hercules/openconfig-hercules-platform-linecard.yang \    
hercules/openconfig-hercules-platform-node.yang \    
hercules/openconfig-hercules-platform-port.yang \    
hercules/openconfig-hercules-platform.yang \    
hercules/openconfig-hercules-qos.yang \    
hercules/openconfig-hercules.yang
```

YANG and Go together
```
bash-4.4# mkdir -p /goSrc
bash-4.4# generator -output_dir=/goSrc -package_name=tutorial demo-port.yang
```

### Understanding YANG-enabled transport protocols

Next, we will use a gNMI client CLI to read the all of the configuration from the Stratum switche leaf1 in our Mininet network:
```
$ util/gnmi-cli --grpc-addr localhost:50001 get /
```

We can rerun the command, but this time pipe the output through the converter utility (then pipe that output to less to make scrolling easier):
```
$ util/gnmi-cli --grpc-addr localhost:50001 get / | util/oc-pb-decoder | less
```

Schema-less representation by requesting the configuration port between leaf1 and h1a:
```
$ util/gnmi-cli --grpc-addr localhost:50001 get \ /interfaces/interface[name=leaf1-eth3]/config
```

Next, we will subscribe to the ingress unicast packet counters for the interface on leaf1 attached to h1a (port 3):
```
$ util/gnmi-cli --grpc-addr localhost:50001 \
--interval 1000 sub-sample \     
/interfaces/interface[name=leaf1-eth3]/state/counters/in-unicast-pkts
```

### Tests
In another window, open the Mininet CLI and start a ping:
```
$ make mn-cli
mininet> h1a ping h1b
```

Finally, we will monitor link events using gNMI's on-change subscriptions.
Start a subscription for the operational status of the first switch's first port:
```
$ util/gnmi-cli --grpc-addr localhost:50001 sub-onchange \    /interfaces/interface[name=leaf1-eth3]/state/oper-status
```

You should immediately see a response which indicates that port 1 is UP.

In the shell running the Mininet CLI, let's take down/up the interface on leaf1 connected to h1a:
```
mininet> sh ifconfig leaf1-eth3 down / up
```

We can also use gNMI to disable or enable an interface.
In a third window, we will use the gNMI CLI to change the configuration value of the enabled leaf from true to false.
```
$ util/gnmi-cli --grpc-addr localhost:50001 set \
    /interfaces/interface[name=leaf1-eth3]/config/enabled \
    --bool-val false / true
```

## Using ONOS as the Control Plane
### Start ONOS

Run the following command in a new terminal window to access the ONOS CLI. Use password rocks when prompted:
```
$ make onos-cli
```

If you see the following error, then ONOS is still starting; wait a minute and try again.
```
ssh_exchange_identification: Connection closed by remote host
make: *** [onos-cli] Error 255
```

When you see the Password prompt, type the default password: rocks. Then type the following command in the ONOS CLI to show the list of running apps:
```
onos> apps -a -s
```

### Build app and register pipeconf

To build the ONOS app (including the pipeconf), run the following command in the second terminal window:
```
$ make app-build
```

Use the following command to load the app into ONOS and activate it:
```
$ make app-reload
```

Alternatively, you can show the list of registered pipeconfs using the ONOS CLI (make onos-cli) command:
```
onos> pipeconfs
```

### Push netcfg to ONOS

On a terminal window, type:
```
$ make netcfg
```

Check the ONOS log:
```
$ make onos-log
```

### Use the ONOS CLI to verify the network configuration

Enter the following command to verify the network config:
```
onos> netcfg
```

Verify that all 4 devices have been discovered and are connected:
```
onos> devices -s
```

Check port information, obtained by ONOS by performing a gNMI Get RPC for the OpenConfig Interfaces model:
```
onos> ports -s device:spine1
```

Check port statistics, also obtained by querying the OpenConfig Interfaces model via gNMI:
```
onos> portstats device:spine1
```

Check the ONOS flow rules. You should see three flow rules for each device. For example, to show all flow rules installed so far on device leaf1:
```
onos> flows -s any device:leaf1
```

To show all groups installed so far, you can use the groups command. For example to show groups on leaf1:
```
onos> groups any device:leaf1
```

### Visualize the topology on the ONOS web UI

Using the ONF Cloud Tutorial Portal, access the ONOS UI:
http://127.0.0.1:8181/onos/ui
Use the username **onos** and password **rocks**.

