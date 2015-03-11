If you don't know what Weave is, read about it [here](https://github.com/zettio/weave).

_NOTE:_ This is a fork of [@lukebond's coreos-vagrant variant that added in weave](https://github.com/lukebond/coreos-vagrant-weave).  I have taken it and added support for Microsoft Azure, Digital Ocean, Google Compute Engine, AWS and got VMWare to work, as well as made it dynamically scalable.  So when you are done, you get a clean N-node setup of clustered CoreOS plus Weave and it will run on top of either virtual-box, VMWare Fusion, VMWare Workstation, AWS, Google Compute Engine, Digital Ocean or Microsoft Azure!  I need to update the readme to provide google compute instructions, but you can see the env vars it is looking for at the end of the Vagrantfile.  More documentation Updates soon.

_NOTE2 (From previous fork):_ This post borrows directly from [@errordeveloper's piece on the Weave Blog](http://weaveblog.com/2014/10/28/running-a-weave-network-on-coreos/) (I suggest you read that first). All I've done is saved you half an hour of rejigging the CoreOS Vagrant repository to incorporate the required changes and removed the ping example. What this gives you is a clean slate of CoreOS with Weave installed and ready to use. If you're used to using the stock CoreOS Vagrant cluster, it is now "plus Weave"! 

## Installation for default 3-node setup on AWS
I've forked the CoreOS Vagrant Weave repository and made the requisite changes. So -  create an AWS Access Key, setup your AWS environment variables, and vagrant up (for a 3-node virtualbox cluster, or vagrant up --provider=aws for a 3-node cluster on AWS.  If you have the Vmware vagrant plugins installed you can do vagrant up --provider=vmware_fusion or vagrant up --provider=vmware_workstation.  You should have the VMWare or Virtualbox plugins installed prior to following these instructions. The discovery tokens are updated dynamically at every "vagrant up".  If you want to run a cluster larger (much larger?) than 3 - just edit the num_instances variable in config.rb, save and follow the directions below.  I've tested it to 20 nodes on AWS.
```
$ git clone https://github.com/stlalpha/coreos-vagrant-gonkulator.git
$ cd coreos-vagrant-gonkulator
$ vagrant plugin install vagrant-aws
$ vagrant box add dummy https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box
```
You need to create a myawsvars_file or add them to your running environment, continue below:
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
Last login: Mon Feb 16 23:29:30 2015 from Q.Q.Q.Q
CoreOS stable (557.2.0)
core@ip-X-X-X-X ~ $ sudo weave expose 172.99.0.1/24
core@ip-X-X-X-X ~ $ logout
```
now node 2
```
$ vagrant ssh core-02
Last login: Mon Feb 16 23:29:30 2015 from Q.Q.Q.Q
CoreOS stable (557.2.0)
core@ip-X-X-X-X ~ $ sudo weave expose 172.99.0.2/24
core@ip-X-X-X-X ~ $ logout
```
And finally node 3 - and verify ping connectivity:
```
$ vagrant ssh core-03
Last login: Mon Feb 16 23:29:30 2015 from Q.Q.Q.Q
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

## Installation for default 3-node setup on Digital Ocean
Generate a rsa keypair for digital ocean.:
```
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/jm/.ssh/id_rsa): /tmp/keypairs/digital_ocean_rsa
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /tmp/keypairs/digital_ocean_rsa.
Your public key has been saved in /tmp/keypairs/digital_ocean_rsa.pub.
The key fingerprint is:
b9:fb:9f:21:e0:33:e1:59:59:00:6a:d2:13:d3:e5:7b jm@foo.host
The key's randomart image is:
+--[ RSA 2048]----+
|      ...   .    |
|     . . o o     |
|    . . . o .    |
|   O . . +   .   |
|    . + S o . E  |
|     . * o   .   |
|      o = . .    |
|         + . o   |
|        ....o    |
+-----------------+
$
```
Clone the repository and install the necessary plugin:
```
$ git clone https://github.com/stlalpha/coreos-vagrant-gonkulator.git
$ cd coreos-vagrant-gonkulator
$ vagrant plugin install vagrant-digitalocean
$ vagrant box add digital_ocean https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box
```
Login to the Digital Ocean console, click on Apps & API, and under Personal Access Tokens, GENERATE NEW TOKEN - and you want it to be read and write.
That token value gets stored as DO_TOKEN below.

You need to create a mydigitaloceanvars_file or add them to your running environment, continue below:
```
$ cat >> mydigitaloceanvars_file << _SCRIPT_
export DO_OVERRIDE_KEY='/tmp/keypairs/digital_ocean_rsa'
export DO_SIZE='1GB'
export DO_REGION='nyc3'
export DO_IMAGE='coreos-stable'
export DO_TOKEN='TOKEN VALUE YOU GENERATED ABOVE'
_SCRIPT_

```
There are available several DO_REGIONS available, but this setup only uses those that support private networking and user-data.  Those are:

New York 3 - nyc3 
Singapore 1 - sgp1
London 1 - lon1

Size definitions are variable - check out the "CREATE DROPLET" screen from the console to see your options and costs.

Possible values for DO_SIZE are:

512MB
1GB
2GB
4GB
8GB
16GB

(DO also has 32, 48 and 64gb options but my account isnt enabled for them)

The example mydigitaloceanvars_file above will create 1GB/1CPU/30GBSSD coreos-stable nodes in the San Francisco 1 region.

Source the file and vagrant up!
```
$ source ./mydigitaloceanvars_file
$ vagrant up --provider=digital_ocean
Bringing machine 'core-01' up with 'digital_ocean' provider...
Bringing machine 'core-02' up with 'digital_ocean' provider...
Bringing machine 'core-03' up with 'digital_ocean' provider...
==> core-01: Using existing SSH key: Vagrant
==> core-01: Creating a new droplet...
==> core-01: Assigned IP address: X.X.X.X
==> core-01: Private IP address: 10.0.0.100
==> core-02: Using existing SSH key: Vagrant
==> core-02: Creating a new droplet...
==> core-02: Assigned IP address: Y.Y.Y.Y
==> core-02: Private IP address: 10.0.0.101
==> core-03: Using existing SSH key: Vagrant
==> core-03: Creating a new droplet...
==> core-03: Assigned IP address: Z.Z.Z.Z
==> core-03: Private IP address: 10.0.0.102
$ vagrant ssh core-01
CoreOS stable (557.2.0)
core@core-01 ~ $ docker ps
CONTAINER ID        IMAGE                           COMMAND                CREATED             STATUS              PORTS                                            NAMES
e6d754450c7e        zettio/weave:git-b76e97ac2426   "/home/weave/weaver    32 seconds ago      Up 31 seconds       0.0.0.0:6783->6783/udp, 0.0.0.0:6783->6783/tcp   weave               
core@core-01 ~ $ fleetctl list-machines
MACHINE		IP		METADATA
611f88d9...	X.X.X.X	-
839370e7...	Y.Y.Y.Y	-
f3e73c24...	Z.Z.Z.Z	-
core@core-01 ~ $ sudo weave status
weave router git-b76e97ac2426
Encryption off
Our name is 7a:2d:bd:d6:86:93 (core-01)
Sniffing traffic on &{10 65535 ethwe 3e:e2:79:a5:5b:4b up|broadcast|multicast}
MACs:
3e:e2:79:a5:5b:4b -> 7a:2d:bd:d6:86:93 (2015-03-04 23:43:26.561169946 +0000 UTC)
4e:66:9f:29:71:63 -> 7a:2d:bd:d6:86:93 (2015-03-04 23:43:27.153323746 +0000 UTC)
7a:2d:bd:d6:86:93 -> 7a:2d:bd:d6:86:93 (2015-03-04 23:43:27.649879851 +0000 UTC)
8a:58:09:bd:17:2d -> 7a:a3:c7:ba:ea:76 (2015-03-04 23:44:08.373185724 +0000 UTC)
7a:a3:c7:ba:ea:76 -> 7a:a3:c7:ba:ea:76 (2015-03-04 23:44:09.131688271 +0000 UTC)
ee:63:59:1e:19:d3 -> 7a:8a:dd:8d:cc:90 (2015-03-04 23:44:36.373658433 +0000 UTC)
7a:8a:dd:8d:cc:90 -> 7a:8a:dd:8d:cc:90 (2015-03-04 23:44:37.560840106 +0000 UTC)
Peers:
Peer 7a:a3:c7:ba:ea:76 (core-03) (v6) (UID 12870450473383100439)
   -> 7a:2d:bd:d6:86:93 (core-01) [X.X.X.X:6783]
   -> 7a:8a:dd:8d:cc:90 (core-02) [Y.Y.Y.Y:6783]
Peer 7a:8a:dd:8d:cc:90 (core-02) (v6) (UID 16201675749546345823)
   -> 7a:a3:c7:ba:ea:76 (core-03) [Z.Z.Z.Z:32961]
   -> 7a:2d:bd:d6:86:93 (core-01) [X.X.X.X:6783]
Peer 7a:2d:bd:d6:86:93 (core-01) (v4) (UID 14668477160103522184)
   -> 7a:a3:c7:ba:ea:76 (core-03) [Z.Z.Z.Z:40115]
   -> 7a:8a:dd:8d:cc:90 (core-02) [Y.Y.Y.Y:60400]
Routes:
unicast:
7a:2d:bd:d6:86:93 -> 00:00:00:00:00:00
7a:a3:c7:ba:ea:76 -> 7a:a3:c7:ba:ea:76
7a:8a:dd:8d:cc:90 -> 7a:8a:dd:8d:cc:90
broadcast:
7a:2d:bd:d6:86:93 -> [7a:a3:c7:ba:ea:76 7a:8a:dd:8d:cc:90]
7a:a3:c7:ba:ea:76 -> []
7a:8a:dd:8d:cc:90 -> []
Reconnects:
core@core-01 ~ $
```

Your cluster is complete and online.  You can follow the example above for AWS to play with weave!

## Installation for default 3-node setup on Microsoft Azure
Follow the instructions here to create your keypairs: https://github.com/MSOpenTech/vagrant-azure
(its under "Using openssl (Linux/Mac)
```
$ openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout mycert_key.pem -out mycert.pem
```
Use mycert.pem as certificate_file and mycert_key.pem as private_key_file.

For mgmt_certificate configuration, create a mycert_mgmt.pem using above command. Use it in your VagrantFile. Then convert the mycert_mgmt.pem to mycert_mgmt.cer to upload to azure portal. 
```
$ openssl x509 -inform pem -in mycert_mgmt.pem -outform der -out mycert_mgmt.cer
```
Clone the repository and install the necessary plugin:
```
$ git clone https://github.com/stlalpha/coreos-vagrant-gonkulator.git
$ cd coreos-vagrant-gonkulator
$ vagrant plugin install vagrant-azure
$ vagrant box add azure vagrant box add azure https://github.com/msopentech/vagrant-azure/raw/master/dummy.box
```
Upload the mycert_mgmt.cer to the Azure console

You need to create a myazurevars_file or add them to your running environment, continue below:
```
$ cat >> myazurevars_file << _SCRIPT_
export AZURE_MGMT_CERT='/Users/stlalpha/azurekeys/mycert.pem'
export AZURE_MGMT_ENDPOINT='https://management.core.windows.net'
export AZURE_SUB_ID='<REDACTED - THIS IS YOUR AZURE SUBSCRIBER ID>'
export AZURE_STORAGE_ACCT='gonkulator'
export AZURE_VM_IMAGE='2b171e93f07c4903bcad35bda10acf22__CoreOS-Stable-557.2.0'
export AZURE_SSH_PRIV_KEY='/Users/stlalpha/azurekeys/mycert.pem'
export AZURE_PRIV_KEY='/Users/stlalpha/azurekeys/mycert.pem'
export AZURE_CERT_FILE='/Users/stlalpha/azurekeys/mycert.pem'
_SCRIPT_
``` 
source the file and vagrant up!
```
$ source ./myazurevars_file
$ vagrant up --provider=azure

==> core-03: Determining OS Type By Image
==> core-02: OS Type is Linux
==> core-03: OS Type is Linux
==> core-01: OS Type is Linux
==> core-03: Attempting to read state for core-03 in gonkulator
==> core-01: Attempting to read state for core-01 in gonkulator
==> core-02: Attempting to read state for core-02 in gonkulator
==> core-01: {:vm_name=>"core-01", :vm_user=>"core", :image=>"2b171e93f07c4903bcad35bda10acf22__CoreOS-Stable-557.2.0", :password=>"", :location=>"West US"}
==> core-01: {:cloud_service_name=>"gonkulator", :storage_account_name=>"gonkulator", :deployment_name=>"core-01", :private_key_file=>"/Users/stlalpha/azurekeys/mycert.pem", :certificate_file=>"/Users/stlalpha/azurekeys/mycert.pem", :ssh_port=>9001, :vm_size=>"Small"}
==> core-02: {:vm_name=>"core-02", :vm_user=>"core", :image=>"2b171e93f07c4903bcad35bda10acf22__CoreOS-Stable-557.2.0", :password=>"", :location=>"West US"}
==> core-02: {:cloud_service_name=>"gonkulator", :storage_account_name=>"gonkulator", :deployment_name=>"core-02", :private_key_file=>"/Users/stlalpha/azurekeys/mycert.pem", :certificate_file=>"/Users/stlalpha/azurekeys/mycert.pem", :ssh_port=>9002, :vm_size=>"Small"}
==> core-03: {:vm_name=>"core-03", :vm_user=>"core", :image=>"2b171e93f07c4903bcad35bda10acf22__CoreOS-Stable-557.2.0", :password=>"", :location=>"West US"}
==> core-03: {:cloud_service_name=>"gonkulator", :storage_account_name=>"gonkulator", :deployment_name=>"core-03", :private_key_file=>"/Users/stlalpha/azurekeys/mycert.pem", :certificate_file=>"/Users/stlalpha/azurekeys/mycert.pem", :ssh_port=>9003, :vm_size=>"Small"}
ResourceNotFound : The hosted service does not exist.
==> core-01: Add Role? - false
Creating deploymnent...
Creating cloud service gonkulator.
Uploading certificate to cloud service gonkulator...
# # succeeded (200)
Storage Account gonkulator already exists. Skipped...
Deployment in progress...
# # # # # # succeeded (200)
==> core-01: Attempting to read state for core-01 in gonkulator
==> core-02: Add Role? - true
==> core-01: VM Status: RoleStateUnknown
==> core-01: Waiting for machine to reach state ReadyRole
==> core-01: Attempting to read state for core-01 in gonkulator
==> core-01: VM Status: RoleStateUnknown
Storage Account gonkulator already exists. Skipped...
Deployment exists, adding role...
Deployment in progress...
# # # # # succeeded (200)
<SNIPPED FOR BREVITY>
$ 
```
When returned to the prompt you can now ssh into your nodes:
```
$ vagrant ssh core-01
==> core-01: Attempting to read state for core-01 in gonkulator
==> core-01: VM Status: ReadyRole
==> core-01: Attempting to read state for core-01 in gonkulator
==> core-01: VM Status: ReadyRole
==> core-01: Looking for local port 22
==> core-01: Found port mapping 9001 --> 22
Last login: Wed Mar 11 03:26:13 2015 from Q.Q.Q.Q
CoreOS stable (557.2.0)
core@core-01 ~ $ fleetctl list-machines
MACHINE		IP		METADATA
08712b3d...	100.112.248.54	-
4b80f75d...	100.112.248.184	-
99acaa43...	100.112.248.189	-
core@core-01 ~ $ sudo weave status
weave router git-b76e97ac2426
Encryption off
Our name is 7a:dd:14:7f:fa:c0 (core-01)
Sniffing traffic on &{9 65535 ethwe da:c3:bf:5c:83:9d up|broadcast|multicast}
MACs:
ba:52:60:c9:c0:7d -> 7a:8f:80:b0:28:ea (2015-03-11 03:25:17.105702699 +0000 UTC)
7a:8f:80:b0:28:ea -> 7a:8f:80:b0:28:ea (2015-03-11 03:25:18.204450699 +0000 UTC)
da:c3:bf:5c:83:9d -> 7a:dd:14:7f:fa:c0 (2015-03-11 03:24:49.088009699 +0000 UTC)
7a:dd:14:7f:fa:c0 -> 7a:dd:14:7f:fa:c0 (2015-03-11 03:24:49.255515199 +0000 UTC)
6e:ad:a6:6e:2a:10 -> 7a:dd:14:7f:fa:c0 (2015-03-11 03:24:50.311954699 +0000 UTC)
86:47:0d:c6:10:3a -> 7a:ec:ea:d9:49:cb (2015-03-11 03:25:17.104174899 +0000 UTC)
Peers:
Peer 7a:dd:14:7f:fa:c0 (core-01) (v4) (UID 9213908346247876447)
   -> 7a:ec:ea:d9:49:cb (core-02) [100.112.248.189:57467]
   -> 7a:8f:80:b0:28:ea (core-03) [100.112.248.54:57388]
Peer 7a:ec:ea:d9:49:cb (core-02) (v4) (UID 18005557667037018474)
   -> 7a:dd:14:7f:fa:c0 (core-01) [100.112.248.184:6783]
   -> 7a:8f:80:b0:28:ea (core-03) [100.112.248.54:6783]
Peer 7a:8f:80:b0:28:ea (core-03) (v4) (UID 3887254487855551153)
   -> 7a:dd:14:7f:fa:c0 (core-01) [100.112.248.184:6783]
   -> 7a:ec:ea:d9:49:cb (core-02) [100.112.248.189:52258]
Routes:
unicast:
7a:ec:ea:d9:49:cb -> 7a:ec:ea:d9:49:cb
7a:8f:80:b0:28:ea -> 7a:8f:80:b0:28:ea
7a:dd:14:7f:fa:c0 -> 00:00:00:00:00:00
broadcast:
7a:8f:80:b0:28:ea -> []
7a:dd:14:7f:fa:c0 -> [7a:ec:ea:d9:49:cb 7a:8f:80:b0:28:ea]
7a:ec:ea:d9:49:cb -> []
Reconnects:
core@core-01 ~ $ 
```
Your cluster is complete and online.  You can follow the example above for AWS to play with weave!
