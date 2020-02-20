# factom_testnet

This probably still needs some additional commenting and refining; additionally, this is specific to servers I've spun up so pay attention to/the/path

*Disclaimer*
This is meant to supplement documentation @ http://www.factom-testnet.com/Introduction
I've found the documentation there to be incomplete. However, there are several things I
don't include in this documentation, namely user setup/permissioning. You may run into
issues that relate to your ability to open/download/run files that can be resolved
by changing the permissions of the file or user.


*Initializing the server:*
This is specifically written for Ubuntu 18.04
For a federated server running the full instance, you'll want 4cores/8GB, if you're just running a follower node, 2 cores, 8GB will be ok.
You'll need to make sure you have 40-60GB of HD space as well.
Configure the security groups before spinning up as well (in our console, security wizard-1 is configured correctly)

Inbound Rules:

TCP port 2376 open to 54.171.68.124 for secure Docker engine communication

2222 to 54.171.68.124, which is the SSH port used by the ssh container

8088 to 54.171.68.124, the factomd API port

8090 to 0.0.0.0, the factomd Control panel, Keeping this open to the world is beneficial on testnet for debugging purposes

8110 to 0.0.0.0, the factomd testnet port

Update and install required packages:
```
sudo apt-get update -y && sudo apt-get upgrade -y
sudo apt-get install curl git apt-transport-https ca-certificates curl software-properties-common -y
```
Add the docker-ce repository and install:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get install docker-ce
```
Add docker privileges to the current user:
```
sudo usermod -aG docker $USER
```
***Configuring Docker***

*Pay attention to the directory*
Make sure you store the Docker Swarm mainnet key and certificate on your system. 
```
sudo mkdir -p /etc/docker
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-testnet-toolkit/master/tls/cert.pem -O /etc/docker/factom-mainnet-cert.pem
sudo wget https://raw.githubusercontent.com/FactomProject/factomd-testnet-toolkit/master/tls/key.pem -O /etc/docker/factom-mainnet-key.pem
sudo chmod 644 /etc/docker/factom-mainnet-cert.pem
sudo chmod 440 /etc/docker/factom-mainnet-key.pem
sudo chgrp docker /etc/docker/*.pem
```

Create a json file, this will help start factomd through docker:

```sudo nano /etc/docker/daemon.json```
Paste this in the file (ensure path is correct!)
```
{
  "tls": true,
  "tlscert": "/etc/docker/factom-mainnet-cert.pem",
  "tlskey": "/etc/docker/factom-mainnet-key.pem",
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock", "fd://"]
}
```


configuring dockerd (Test method, only use this if json file works incorrectly)
```
sudo dockerd -H=unix:///var/run/docker.sock -H=0.0.0.0:2376 --tls --tlscert=/etc/docker/factom-mainnet-cert.pem --tlskey=/etc/docker/factom-mainnet-key.pem
```


Service File Override creates override file, saves as default, this should ensure docker/factomd autmatically starts
```
sudo systemctl edit docker.service
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
```
then reload
```	
sudo systemctl daemon-reload
sudo systemctl restart docker.service
```
If you need to restart
```
systemctl stop docker.service
systemctl stop docker.socket
systemctl daemon-reload
systemctl start docker.service
systemctl status docker.service docker.socket
```

moving old database
This may save you time syncing the ledger, we may want to make this sexy and build an s3 bucket that holds the ledger which we can pull from everytime we spin up a new node (I don't know if s3 is the right terminology)
```
sudo cp -r /data/var/lib/docker/volumes/factom_database/_data /var/lib/docker/volumes/factom_database/_data
```

Create the FactomD volumes to hold your identity keys and the database 
```
docker volume create factom_database
docker volume create factom_keys
```
Need to create factom.conf file, this will initialize the factom docker instance when it starts:

This line pulls the .config file
```

sudo curl https://raw.githubusercontent.com/FactomProject/factomd-testnet-toolkit/master/factomd.conf.EXAMPLE -o /var/lib/docker/volumes/factom_keys/_data/factomd.conf

```

#Join the Factom Testnet Swarm:
```
docker swarm join --token SWMTKN-1-0bv5pj6ne5sabqnt094shexfj6qdxjpuzs0dpigckrsqmjh0ro-87wmh7jsut6ngmn819ebsqk3m 54.171.68.124:2377
```

*Start the FactomD container (pay attention to the factomd version, as of 11/20/18 is v6.0.1-alpine)*
```
docker run -d --name "factomd" -v "factom_database:/root/.factom/m2" -v "factom_keys:/root/.factom/private" -p "8088:8088" -p "8090:8090" -p "8110:8110" -l "name=factomd" factominc/factomd:v6.1.0-rc1-alpine -broadcastnum=16 -network=CUSTOM -customnet=fct_community_test -startdelay=600 -faulttimeout=120 -config=/root/.factom/private/factomd.conf
```

The below command will give you your node ID so that other peers can find you in the network/
```
docker info | grep NodeID
```

***This will remove factom_database if you need to start over rebuilding the ledger***
```
sudo docker volume rm -f factom_database
```

# Updating Factomd (need to pay attention to testnet name):
```
sudo docker stop factomd
sudo docker rm factomd
sudo docker run -d --name "factomd" -v "factom_database:/root/.factom/m2" -v "factom_keys:/root/.factom/private" -p "8088:8088" -p "8090:8090" -p "8110:8110" -l "name=factomd" factominc/factomd:v6.2.3-rc3-alpine -broadcastnum=16 -network=CUSTOM -customnet=fct_community_test -startdelay=600 -faulttimeout=120 -config=/root/.factom/private/factomd.conf
```
Once you're running the factomd d instance, you may want to monitor the ledger sync:
```
docker logs factomd --tail "10"
```

the linux command ```df``` will show you available space on your server
```top``` or (preferable but you'll have to download it) ```htop``` will show running services

# Setting Up Server Identity
*Install Factom wallet and factomd-cli*
Go to: https://github.com/FactomProject/distribution
to get the latest releases of both factom-cli and walletd

to download dependency files:
```
sudo wget https://path/to/latest releases.dep
```
Once downloaded, for both files:
```
sudo dpkg -i 'current.factom.release.dep'
```
Create a systemctl file to automatically start the wallet when the node is initialized, make sure factom-cli is unpacked first:
```
sudo nano /etc/systemd/system/factom-walletd.service

#insert this code and save/close the file
[Unit]
Description=Run the Factom Wallet service
Documentation=https://github.com/FactomProject/factom-walletd
After=network-online.target

[Service]
User=factom-walletd #This assumes user, if using root, delete this
Group=factom-walletd #This assumes group, if using root, delete this
EnvironmentFile=-/etc/default/factom-walletd
ExecStart=/usr/bin/factom-walletd $FACTOM_WALLETD_OPTS
KillMode=control-group
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
restart the wallet:
```
systemctl daemon-reload
systemctl start factom-walletd
```

Now factom-walletd should be running on localhost:8089, the default port. 
If you want to enable this to start on boot run
```
systemctl enable factom-walletd
```
Get a new factom address:
```
factom-cli newecaddress
```
***Get some testnet factoids:***
https://faucet.factoid.org/

Make sure your wallet's been funded:
```
factom-cli balance ECXXXXXXXXXXXXXX
```
Export your address:
```
factom-cli exportaddresses
```
Take note of your private TC address for the next step, beginning with "Esxxxxxxxxxxxxxxxx

*Download and build server identity program.*
The following are directions for linux.
Install git, golang 1.10, and glide. Set the proper $GOPATH environment variable.

```
sudo apt-get install git golang-go golang-glide
```

Edit ~/.profile and add these lines to the bottom (nano):
```
GOPATH=$HOME/go
PATH=$PATH:$GOPATH/bin
```
type:
```
source .profile
```

Clone the serveridentity program:
```
mkdir -p $GOPATH/src/github.com/FactomProject/
cd $GOPATH/src/github.com/FactomProject/
git clone https://github.com/FactomProject/serveridentity.git
cd serveridentity
glide install
go install
cd signwithed25519
go install
```

# The Tricky Part
You'll need to use your wallet's private key as well as the serveridentity and signwithed22519
to create your server identity and generate the private keys for your server. This is best done offline
in a secure facility with two factor entry authentication and a blind guy who knows kong-fu. If you don't
have access to something like that, you can use VirtualBox to spin up a virtual linux instance to run the files:

https://www.virtualbox.org/

You'll need to get images to run your linux virtual image:

https://www.osboxes.org/virtualbox-images/

You'll also need winscp to transfer files between your computer and the server:

https://winscp.net/eng/download.php

If you're on a Mac:
https://cyberduck.io/

Once you have all that, use winscp to copy the serveridentity and signwithed25519 to your desktop, then into your virtualbox drive

Once in your virtual linux instance, run, entering in your private TC address from earlier:
```
serveridentity full elements Esxxxxxxxxxxxxxxxxxxxxx -n=important -f
```
***This will give you 4 secret keys, write them down, do not lose these, do not digitally store them, do not THINK about them***

This should also create two files important.conf and important.sh, they should be about 1kb and 4kb in size.
If not, rerun the above command.

Now, time to reverse the process.
Take important.conf and important.sh and use winscp to upload them to your factomd testnet server.
Edit the important.sh file (using nano) and add ```-s=localhost:8088``` right after ```factom-cli``` every time it appears.
Now run important.sh. ```sudo sh important.sh``` You should see a bunch of hashes, within 10 minutes the identifcation should be
written to the blockchain. 

You'll also need to modify your factomd.conf file with information from important.conf and restart factomd.

factomd.conf should be located here: /var/lib/docker/volumes/factom_keys/_data/factomd.conf

Take all information from important.conf, the file should look something like:
```
[app]
chainID 88888xxxxxxxx
............
...........
```

Copy all information (including [app]) to the end of the factomd.conf file replacing the default chainID, which looks something like this: ```fa1e0000000000000000```

Restart docker per the *Updating Factomd* code above 

You can check here to see your identity/chainID, just search for your public wallet address (ECxxxxxx....):

https://testnet.factoid.org/data?type=dblock-list

# Oh, you thought we were done?

We need to set our efficiency and coinbase address (coinbase refers to the Factoid address not the website). Before you can do this, you'll need to be admitted as a authority node, one of the Factom Admins can do this.

You'll need to be authorized onto the testnet or mainnet before this happens

First make sure you have node.js and npm installed, if not run:
```
curl -sL https://deb.nodesource.com/setup_11.x | sudo -E bash -
#after download is complete:
sudo apt-get install -y nodejs
```
Now install factom-identity-cli:
```
sudo npm install -g factom-identity-cli
```
Now for the next confusing part, you'll need your FCT address, your chainID, your management ID, and your level 1 secret from earlier (when you ran serveridentity)

if you don't have an FCT address/coins, just run ```factom-cli newfctaddress```, go to the faucet, and input the address

If you don't know the management ID, go to: http://136.144.191.225:8090 and search for your entry using your chainID, scroll down, the management ID is the second block with header Register Server Management starts with 88888

*Update coinbase address*

#Parameters: <Identity root chain ID> <FCT public address> <SK1 private key> <Paying private EC address>

```factom-identity-cli update-coinbase-address -s localhost:8088 .....```

*Update efficiency*

#Parameters: <Identity root chain ID> <Efficiency> <SK1 private key> <Paying private EC address>

 ```factom-identity-cli update-efficiency -s localhost:8088 ...```
 
 To see what your current efficieny and coinbase is:
 
 ```factom-identity-cli get -s localhost:8088 <ChainID>```
 
 
# Brainswap
Brainswapping is the idea of having 2 nodes switch identities at the same time. We call is "brainswapping" because a node's identity dictates how it behaves.

Note: The procedure doesn't actually have to be a "swap"; a "brain-transfer" is also an alternative.

It was implemented as a way to update the network without having to bring it down, as federated servers can "brainswap" with standby nodes that have already been updated with the new code.

The network will not perceive this as a node going offline, as the identity (and thus the associated federated server) is still online.

After transferring the Authority identity the node operator can now shutdown their original node, perform necessary updates, bring it back online and finally brainswap the identity back into the original server.

The procedure can also be used for migrating the Authority identity to a new physical server, by not performing the brainswap a second time to reverse the first swap.

Definitions
Federated Node is the node that holds your authority identity, and needs to be updated.
Standby Node is a follower node on the network that you control.
Prepping the Brain Swap
To perform the brain swap you will need 1 standby node that is ready. The standby node should be running the most recent Factomd software version.

Determining if you Standby Node is ready
If your standby node is not in sync with the network, performing the brainswap will result in your authority server going offline, so it is crucial to first check the health of the standby node.

Check if the DBHeight matches that of the network
Check if the minutes are following that of the network
Check the process list for <nil>, that indicates some network instability (Procedures for the above described at the bottom of this document)
Performing the Brain Swap (Read through before performing)
Once your standby node is ready, we can prep for the swap. The swap requires both config files (located on Federated Node and Standby Node) to be modified.

Open both config files in parallel in a text editor of your choice. (Procedures for editing is described at the bottom of this document)

Swap the following lines in the two config files:

IdentityChainID	                      = FA1E000000000000000000000000000000000000000000000000000000000000
LocalServerPrivKey                = 4c3xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
LocalServerPublicKey            = cc1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
(If you prefer to do a brain-transfer instead of a swap, you can just comment out the lines in your Federated node by placing a ; in front of these lines.)

Then add an additional line:

ChangeAcksHeight                      = 0
This additional line is the brainswap logic. You will want to set the ChangeAcksHeight to some block height in the future (remember the block height on the control panel is the last saved height, not the current working height!).

The safe height is the one you see in the control panel + 4. (localhost:8090). If you know how to read the more detailed page, you can get away with a closer number.

Once you set the ChangeAcksHeight to DBHeight+4, save both files.

At the block height ChangeAcksHeight you should see both nodes change identities. If none of your nodes crash, and the identities change, the swap was successful.
