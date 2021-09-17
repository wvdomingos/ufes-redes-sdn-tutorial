# System requirements

**Option 1 - Download tutorial VM**
Use the following link to download the VM (4 GB): http://bit.ly/ngsdn-tutorial-ova

**Option 2 - Manually install Docker and other dependencies**
All exercises can be executed by installing the following dependencies:
* Docker v1.13.0+ (with docker-compose)
* Make
* Python 3
* Bash-like Unix shell
* Wireshark (optional)

### Get this repo

To work on the exercise we will need to clone this repo
```
$ cd ~
$ git clone -b advanced https://github.com/opennetworkinglab/ngsdn-tutorial
``` 

Download / upgrade dependencies
```
# cd ~/ngsdn-tutorial
# make deps
```

### Repo structure

This repo is structured as follows:
* p4src/ - P4 implementation
* yang/ - Yang model 
* app/ - custom ONOS app Java implementation
* mininet/ - Mininet 
* util/ - Utility scripts
* ptf/ - P4 data plane unit tests based on Packet Test Framework (PTF)

**Click on the link to see the instructions:**
* [Exercises](EXERCISES.md)
