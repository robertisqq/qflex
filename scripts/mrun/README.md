# Multinode
0. [Prerequisites](#Prerequisites)
1. [Introduction](#Introduction)
  1. [Features](#Features)
2. [Usage](#Usage)
  1. [mrun](#mrun)
  1. [Options](#Options)
3. [Tutorial](#Tutorial)

## Prerequisites
* python 2.7
* python psutil
* ns3
  * bridg/tap

__Ubuntu__
```
sudo apt-get install -y python python-pip bridge-utils uml-utilities
sudo pip install psutil
cd qflex/3rdparty/ns3 && git checkout parsa_ns3
./waf configure --build-profile=optimized --enable-examples && ./waf

```

## Introduction
Multinode was wrritten to allow for running distributed VM systems. By connecting multiple QEMU instances together and using user-level networking (i.e. NS3), Multinode provides a workflow for observing scale-out workloads.

### Features
Multinode scripts have been written in a way to allow for flexible configuration of each connected node as well as interacting with each one of them.

Multinode itself runs as the __MASTER__ of all the nodes and through its interface, it allows for interacting with each noth node individually as well as collectively when neccessary. for example saving/loading all node states together is one of the collective functionalities.

Multinode can lockstep-run its nodes for a more deterministic behavior.

## Usage

### mrun
mrun is the entry-level script that users will use to configure their setup. The heart of mrun relies on parsing a single setup file (XML) which has paths to instance files (XML) nested inside of it. mrun parses the former and then fires up all the instances sequentially.

### Options
`-h, --help`            
Show this help message and exit

`-g, --generate`       
generate sample files for use with this script.
i.e. qemu_instance_sample_file and qemu_setup_sample_file

`-d, --debug`           
generate sample files for use with this script.
i.e. qemu_instance_sample_file and qemu_setup_sample_file

`-o, --output`        
Output the commands used by this script for running the instances. useful for debugging purposes

`-c INPUT FILE/TEXT OUTXML, --convert INPUT FILE/TEXT OUTXML`            
Convert the text provided into a XML instance file. this options takes two arguments. the first is the text you wish to use - please use quotes for defining your command as spaces are prone to causing misunderstandings. the second is your output file.

`-r FILE, --run FILE`   
run multinode emulation using the setup file provided

`-n IP:PORT, --mnet IP:PORT`                        
used in conduction with multi-node network. sets up host

`-ns ns-script file`
Setup NS3 network by passing in the location of your ns3 source file- e.g. tap-csma-virtual-machine-parsa.cc.
NOTE: NS3 shold be compiled with our PARSA script beforehand. mrun will take care of setting up all the options for instances - so no need for providing them.
also, mrun will setup all the networks required. so no need to do that as well. setting up networks yourself
might cause conflict with using this option.

`-qmp`                  
used in conjucnction wih the -r command for enabling qmp.
i will use the qmp option provided in instance files if any available. if no qmp option provided, this script will generate one for each instance.

`-q QUANTUM, --quantum QUANTUM`
used in conjucnction wih the -r command for setting a quantum which will be used to determine the desired quantum lock-step execution - this value represents the number of instruction that each qemu executes
before switching to the next instance.

`-lo LOAD, --load LOAD`                        
Loads the snapshot provided. it will automatically append the instance name for instance specific load.
NOTE: this uses loadext only

`-u, --update-file`
This option can be used for updating your instance file. as this scripts can add to it and you might want to
see what the script has done to your file. useful for debugging

`-l {NOTSET,DEBUG,INFO,WARNING,ERROR,CRITICAL}, --log {NOTSET,DEBUG,INFO,WARNING,ERROR,CRITICAL}`
Logging events

using QMP (QEMU Machine Protocol) - https://wiki.qemu.org/Documentation/QMP
QMP is the only QEMU API that is considered stable. so we try to use it to our benefit. when the option -qmp is specified. mrun will look for the qmp option within each instance and will setup a connection to each one of them. use the help command to get an idea for how mrun works.

NS3 for Multi-node
we use NS3 for the connection between nodes. there is an NS3 folder within the cloned Qflex directory that should be paste on top of your NS3 directory. use this link to setup NS3. - https://www.nsnam.org/docs/tutorial/html/getting-started.html build with examples. we use the Tap/Bridge model to set up an NS3 communication.

###  known issues
kernel CMD line option will result in QEMU hanging midway through boot . i.e. when you do -append 'console=XXX' since mrun Pipes stdio for the process.

### Tutorial
To run Multinode, you need to follow the steps below:

* Use `mrun -g` to generate sample files. for a 2-node setup, duplicate the instance-file and fill out all the xml files. it should be quiet straight forward.

* once your files are all set, you can execute `mrun -r <setup file>` to start. if you have not provided the `-qmp` option, you will not see a prompt to interact with the nodes. so in order to do so, you can either manually edit the instance files and provide the QMP option manually or just use mrun's `-qmp` option.
