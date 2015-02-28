If you don't know what Weave is, read about it [here](https://github.com/zettio/weave).

Test

_NOTE:_ This is a fork of [@lukebond's coreos-vagrant variant that added in weave](https://github.com/lukebond/coreos-vagrant-weave).  I have taken it and added support for Google Compute Engine, AWS and got VMWare to work, as well as made it dynamically scalable.  So when you are done, you get a clean N-node setup of clustered CoreOS plus Weave and it will run on top of either virtual-box, VMWare Fusion, VMWare Workstation, AWS or Google Compute Engine!  I need to update the readme to provide google compute instructions, but you can see the env vars it is looking for at the end of the Vagrantfile.  Documentation Updates soon.

_NOTE2 (From previous fork):_ This post borrows directly from [@errordeveloper's piece on the Weave Blog](http://weaveblog.com/2014/10/28/running-a-weave-network-on-coreos/) (I suggest you read that first). All I've done is saved you half an hour of rejigging the CoreOS Vagrant repository to incorporate the required changes and removed the ping example. What this gives you is a clean slate of CoreOS with Weave installed and ready to use. If you're used to using the stock CoreOS Vagrant cluster, it is now "plus Weave"! 

## Installation
I've forked the CoreOS Vagrant Weave repository and made the requisite changes. So -  create an AWS Access Key, setup your AWS environment variables, and vagrant up (for a 3-node virtualbox cluster, or vagrant up --provider=aws for a 3-node cluster on AWS.  If you have the Vmware vagrant plugins installed you can do vagrant up --provider=vmware_fusion or vagrant up --provider=vmware_workstation.  You should have the VMWare or Virtualbox plugins installed prior to following these instructions. The discovery tokens are updated dynamically at every "vagrant up".  If you want to run a cluster larger (much larger?) than 3 - just edit the num_instances variable in config.rb, save and follow the directions below.  I've tested it to 20 nodes on AWS.
```
$ git clone https://github.com/stlalpha/coreos-vagrant-gonkulator.git
$ cd coreos-vagrant-gonkulator
$ vagrant plugin install vagrant-aws
$ vagrant box add dummy https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box
```
If you want to setup AWS now - you need to create a myawsvars_file or add them to your running environment, continue below.  If you do not want to setup/run AWS now - just vagrant up <enter>
```
$ cat >> myawsvars_file << _SCRIPT_
export AWS_KEY='KEY FROM AWS CONSOLE'
export AWS_SECRET='SECRET FROM AWS CONSOLE'
exoirt AWS_SECURITY_GROUP='NAME OF VPC SEC GROUP' you need one that allows 22/tcp and 6783/tcp and 6783/udp inbound 
export AWS_KEYNAME='Name of the AWS Keypair you are using'
export AWS_KEYPATH='/path/to/keyfile-for-above-named-pair.pem'
export AWS_AMI='whichever ami you want from https://coreos.com/docs/running-coreos/cloud-providers/ec2/ - make sure its available via the region you select next'
export AWS_REGION='which AWS reagion you want - from aws ec2 describe-regions'
export AWS_INSTANCE='which instance you want - from aws ec2 describe-instances'
export AWS_ACCESS_KEY=$AWS_KEY
export AWS_SECRET_KEY=$AWS_SECRET
_SCRIPT_
```
A good example file would be this one that uses a t1.micro instance, out of the us-west-2 region, running a coreos PV AMI out of the stable channel.  
I left out my security keys for obvious reasons:
```
$ cat myawsvars_file
export AWS_KEY='XXXXXXX'
export AWS_SECRET='XXXXXXXXXXX'
export AWS_SECRET='theweaves'
export AWS_KEYNAME='myenv-key'
export AWS_KEYPATH='/Users/stlalpha/ec2keypairs/myenv-key.pem'
export AWS_AMI='ami-f3702bc3'
export AWS_REGION='us-west-2'
export AWS_INSTANCE='t1-micro'
export AWS_ACCESS_KEY=$AWS_KEY
export AWS_SECRET_KEY=$AWS_SECRET
```
At this point you can now source your variables and launch vagrant on aws:
```
$ source ./myawsvars_file 
$ vagrant up --provider=aws
Bringing machine 'core-01' up with 'aws' provider...
Bringing machine 'core-02' up with 'aws' provider...
Bringing machine 'core-03' up with 'aws' provider...
==> core-02: Warning! The AWS provider doesn't support any of the Vagrant
==> core-02: high-level network configurations (`config.vm.network`). They
==> core-02: will be silently ignored.
==> core-02: Launching an instance with the following settings...
==> core-02:  -- Type: t1.micro
==> core-02:  -- AMI: ami-f3702bc3
<A WHOLE BUNCH OF ETCD OUTPUT>

```
I clipped the above for brevity...it will go on and eventually you will see:
```
==> core-02: Waiting for instance to become "ready"...
==> core-03: Waiting for instance to become "ready"...
==> core-01: Waiting for instance to become "ready"...
==> core-01: Waiting for SSH to become available...
==> core-02: Waiting for SSH to become available...
==> core-03: Waiting for SSH to become available...
==> core-02: Machine is booted and ready for use!
==> core-03: Machine is booted and ready for use!
==> core-01: Machine is booted and ready for use!
$
```
Now you can login to the master node (always named core-01) and verify that the weave router is running:
```
$ vagrant ssh core-01
CoreOS stable (557.2.0)
core@ip-X-X-X-X ~ $ docker ps        
CONTAINER ID        IMAGE                COMMAND                CREATED              STATUS              PORTS                                            NAMES
00c64a527203        zettio/weave:0.9.0   "/home/weave/weaver    About a minute ago   Up About a minute   0.0.0.0:6783->6783/tcp, 0.0.0.0:6783->6783/udp   weave               
core@ip-X-X-X-X ~ $ 
```
And that your weave router can see its peers:
```
core@ip-X-X-X-X ~ $ sudo weave status
weave router 0.9.0
Encryption off
Our name is 7a:24:1d:ba:6e:cb
Sniffing traffic on &{9 65535 ethwe d6:9a:f1:e1:61:ca up|broadcast|multicast}
MACs:
e2:db:65:77:26:a6 -> 7a:c7:9d:70:27:5d (2015-02-16 23:28:27.717047612 +0000 UTC)
6e:6f:2b:c7:db:59 -> 7a:55:fe:eb:be:e0 (2015-02-16 23:28:30.574235677 +0000 UTC)
7a:55:fe:eb:be:e0 -> 7a:55:fe:eb:be:e0 (2015-02-16 23:28:31.20557814 +0000 UTC)
d6:9a:f1:e1:61:ca -> 7a:24:1d:ba:6e:cb (2015-02-16 23:28:08.184755968 +0000 UTC)
42:cd:29:43:3e:0c -> 7a:24:1d:ba:6e:cb (2015-02-16 23:28:08.416928726 +0000 UTC)
7a:24:1d:ba:6e:cb -> 7a:24:1d:ba:6e:cb (2015-02-16 23:28:09.642968486 +0000 UTC)
7a:c7:9d:70:27:5d -> 7a:c7:9d:70:27:5d (2015-02-16 23:28:27.716918115 +0000 UTC)
Peers:
Peer 7a:24:1d:ba:6e:cb (v4) (UID 10753626337984829546)
   -> 7a:c7:9d:70:27:5d [52.10.54.184:60936]
   -> 7a:55:fe:eb:be:e0 [52.10.63.114:33242]
Peer 7a:c7:9d:70:27:5d (v9) (UID 3613930806318167490)
   -> 7a:24:1d:ba:6e:cb [52.10.66.91:6783]
   -> 7a:55:fe:eb:be:e0 [52.10.63.114:6783]
Peer 7a:55:fe:eb:be:e0 (v8) (UID 4208635013132343982)
   -> 7a:24:1d:ba:6e:cb [52.10.66.91:6783]
   -> 7a:c7:9d:70:27:5d [52.10.54.184:37337]
Routes:
unicast:
7a:c7:9d:70:27:5d -> 7a:c7:9d:70:27:5d
7a:55:fe:eb:be:e0 -> 7a:55:fe:eb:be:e0
7a:24:1d:ba:6e:cb -> 00:00:00:00:00:00
broadcast:
7a:55:fe:eb:be:e0 -> []
7a:24:1d:ba:6e:cb -> [7a:c7:9d:70:27:5d 7a:55:fe:eb:be:e0]
7a:c7:9d:70:27:5d -> []
Reconnects:
core@ip-X-X-X-X ~ $
```
So - you can see that you have three peers listed, your coreos-weave cluster is live and ready for testing.
Let's create a new weave vlan on each of the nodes and verify connectivity:
```	
$ vagrant ssh core-01
Last login: Mon Feb 16 23:29:30 2015 from 24.207.140.202
CoreOS stable (557.2.0)
core@ip-X-X-X-X ~ $ sudo weave expose 172.99.0.1/24
core@ip-X-X-X-X ~ $ logout
```
now node 2
```
$ vagrant ssh core-02
Last login: Mon Feb 16 23:29:30 2015 from 24.207.140.202
CoreOS stable (557.2.0)
core@ip-X-X-X-X ~ $ sudo weave expose 172.99.0.2/24
core@ip-X-X-X-X ~ $ logout
```
And finally node 3 - and verify ping connectivity:
```
$ vagrant ssh core-03
Last login: Mon Feb 16 23:29:30 2015 from 24.207.140.202
CoreOS stable (557.2.0)
core@ip-X-X-X-X ~ $ sudo weave expose 172.99.0.3/24
core@ip-X-X-X-X ~ $ ping 172.99.0.1 -c 3
PING 172.99.0.1 (172.99.0.1) 56(84) bytes of data.
64 bytes from 172.99.0.1: icmp_seq=1 ttl=64 time=1.72 ms
64 bytes from 172.99.0.1: icmp_seq=2 ttl=64 time=2.22 ms
64 bytes from 172.99.0.1: icmp_seq=3 ttl=64 time=1.76 ms

--- 172.99.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 1.723/1.904/2.227/0.234 ms
core@ip-X-X-X-X ~ $ ping 172.99.0.2 -c 3
PING 172.99.0.2 (172.99.0.2) 56(84) bytes of data.
64 bytes from 172.99.0.2: icmp_seq=1 ttl=64 time=3.02 ms
64 bytes from 172.99.0.2: icmp_seq=2 ttl=64 time=1.84 ms
64 bytes from 172.99.0.2: icmp_seq=3 ttl=64 time=1.90 ms

--- 172.99.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 1.847/2.257/3.022/0.541 ms
core@ip-X-X-X-X ~ $ ping 172.99.0.3 -c 3
PING 172.99.0.3 (172.99.0.3) 56(84) bytes of data.
64 bytes from 172.99.0.3: icmp_seq=1 ttl=64 time=0.080 ms
64 bytes from 172.99.0.3: icmp_seq=2 ttl=64 time=0.053 ms
64 bytes from 172.99.0.3: icmp_seq=3 ttl=64 time=0.053 ms

--- 172.99.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.053/0.062/0.080/0.012 ms
core@ip-X-X-X-X ~ $ 
```
Boom.  Three nodes, on aws, with a weave VLAN hooked to the coreos host.  Enjoy!
