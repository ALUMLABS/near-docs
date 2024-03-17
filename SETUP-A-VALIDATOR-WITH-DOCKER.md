# SETUP NEAR VALIDATOR WITH DOCKER

__Our tutorials are based on Ubuntu!__

Our tutorials are based on Docker which delivers containerization, improving the overall security of your node.

More info about setting up a NEAR validator on page [Validator Bootcamp](https://near-nodes.io/validator/validator-bootcamp)

---

## SECURE YOUR NEAR VALIDATOR

After having purchased the right server (see [HARDWARE REQUIREMENTS](https://near-nodes.io/validator/hardware)), first follow our tutorial [SECURE YOUR VALIDATOR](https://github.com/ALUMLABS/near-docs/secure-your-validator.md) to harden your server's security.

---

## USE ROOT OR A DEDICATED USER?

As you could have (read: should have) read in the previous step, all of the commands found in our tutorials start with `#` instead of `$ sudo`.

---

## MAKE SURE SWAP IS OFF

To really take advantage of NVMe's multi-channel speed, disable swap:

* Edit fstab: `root@node ~ # nano /etc/fstab`
* Find your swap disk and comment it by placing a `#` in front of the line. This will prevent it from being mounted when you reboot the node in the future.

```
#UUID=f67b4de1-5f0c-4258-8a61-f662db61e224 none swap sw 0 0
```

* Turn off swap: `root@node ~ # swapoff -a`

---

## INSTALL PREREQUISITES: SCREEN + DOCKER + NODE + NPM + NEAR-CLI

```
root@node ~ # apt update && apt upgrade -y
root@node ~ # curl -sL https://deb.nodesource.com/setup_18.x | sh
root@node ~ # apt install screen docker.io nodejs npm -y
root@node ~ # npm -v
root@node ~ # node -v
root@node ~ # npm config set prefix /usr/local
root@node ~ # npm i -g near-cli
```

---

## CREATE FOLDERS AND UPLOAD FILES

* Create these folders: `root@node ~ # mkdir -p /opt/near/node/near/data/`


#### UPLOAD YOUR VALIDATOR_KEY.JSON

* Create file `root@node ~ # nano /opt/near/node/validator_key.json` and upload its contents.

*If you don't have one yet, follow these steps first:* <br />
https://near-nodes.io/validator/validator-bootcamp#step-2--create-a-wallet


#### SCRIPT: RUN-NEAR

* Create file `root@node ~ # nano /opt/near/run-near.sh` and add its contents:
	- Check https://github.com/near/nearcore/releases for the latest stable release and adjust the version inside the script.

```bash
#!/bin/bash
CONT=$(basename $PWD)
IMG=nearprotocol/nearcore
VER=1.37.3
set -x
docker pull $IMG:$VER
docker stop $CONT
docker rm $CONT
docker run --name $CONT -d -v $(pwd)/node:/srv -p24567:24567 -p127.0.0.1:3030:3030 --restart always $IMG:$VER
docker logs -f $CONT --since 1h
```

__DO NOT RUN THE SCRIPT YET!__

For those already familiar with Docker: <br />
You probably noticed we are binding port 3030 to localhost instead of exposing it to the world. If you need the RPC port to be available to the outside, whether it be available to everyone or just one specific IP for your own use, check out our tutorial [SECURE YOUR VALIDATOR](https://github.com/ALUMLABS/near-docs/secure-your-validator.md) again for the appropriate UFW rules.

*Contact us for our custom monitoring script so you can use a service such as Betterstack to monitor your node*


#### SCRIPT: MAKE-ACTIVE

This script stops a running node, moves your validator_key.json from the directory one level up to the nesr directory that's mounted inside the Docker container, and then restarts the node.

* Create file: `root@node ~ # nano /opt/near/make-active.sh`
* Add this: 

```bash 
#!/bin/bash
CONT=$(basename $PWD)
set -x

docker stop $CONT
mv /opt/near/node/validator_key.json /opt/near/node/near/validator_key.json
docker start $CONT
docker logs -f $CONT --since 1h
```

__DO NOT RUN THE SCRIPT YET!__


#### SCRIPT: MAKE-INACTIVE

This script stops a running node, moves your validator_key.json back to the directory one level up, and then restarts the node.

* Create file: `root@node ~ # nano /opt/near/make-inactive.sh`
* Add this: 

```bash
#!/bin/bash
CONT=$(basename $PWD)
set -x

docker stop $CONT
mv /opt/near/node/near/validator_key.json /opt/near/node/validator_key.json
docker start $CONT
docker logs -f $CONT --since 1h
```

__DO NOT RUN THE SCRIPT YET!__

---

## GENERATE UNIQUE NODE KEY

Every node, validator or not, needs to 'tell' the network who they are. Every node should also be unique, which is why we generate a node_key.json per node. If you make your backup node 'inactive' (meaning: not a live validator), it still is an online node contributing to the network based on its own node_key. Without this node_key, your node will not run!

```bash
root@node ~ # echo 'export NEAR_ENV=mainnet' >> ~/.bashrc
root@node ~ # source ~/.bashrc
root@node ~ # near generate-key node
root@node ~ # cp ~/.near-credentials/mainnet/node.json /opt/near/node/near/node_key.json
```

__Don't forget to change `private_key` to `secret_key`!:__

`root@node ~ # sed -i 's/private_key/secret_key/g' /opt/near/node/near/node_key.json`

---

## DOWNLOAD LATEST SNAPSHOT

* Start a screen session: `root@node ~ # screen -S copy`
  - Don't know `screen` yet? Read more here: https://www.tecmint.com/screen-command-examples-to-manage-linux-terminals/
* From your new screen session, go home, clone repo s5cmd, step into it, and build the image: 

```bash
root@node ~ # cd ~
root@node ~ # git clone https://github.com/peak/s5cmd
root@node ~ # cd s5cmd
root@node ~ # docker build -t s5cmd .
```

* Download file `latest` which contains the timestamp of the latest snapshot:
  - Part `~/s5cmd:/root/s5cmd` means: mount `~/s5cmd` into the container at location `/root/s5cmd`
  - Downloading the latest file to docker location `/root/s5cmd` means it will be available to you at location `~/s5cmd/latest`

```bash
root@node ~ # docker run --rm -v ~/s5cmd:/root/s5cmd s5cmd --no-sign-request cp 's3://near-protocol-public/backups/mainnet/rpc/latest' /root/s5cmd
root@node ~ # cat ~/s5cmd/latest
```

* Use the timestamp to start copying the snapshot to your NEAR data folder:
  - Like before, we mount directory `/opt/near/node/near/data/` inside the docker container at `/root/near` which we download the snapshot to

```bash
root@node ~ # docker run --rm -v /opt/near/node/near/data/:/root/near s5cmd --no-sign-request sync 's3://near-protocol-public/backups/mainnet/rpc/$(cat ~/s5cmd/latest)/*' /root/near/
```

* You can detach the screen session with `control+a d` and it will keep downloading the snapshot for a few hours. Come back later to continue. If you want to check the status of the download, either attach the screen session again with `# screen -x` or keep an eye on the directory size `# du -sh /opt/near/node/near/data` which should be around 1.3TB at the time of writing.

---

## FIRST RUN & FINAL CHECK!

Okay, we have everything ready for our first run!

```bash
root@node ~ # cd /opt/near
root@node ~ # bash run-near.sh
```

* You should see a lot of logs now as the final line of file `run-near.sh` says: `docker logs -f near --since 1h`
* Control+c to stop tailing the logs (don't worry, you won't stop the node)
  - To tail it again, just run this again: `docker logs -f near --since 1h`
* Or check status via RPC:

```
root@node ~ # curl http://localhost:3030/status
```

---

## WHAT NOW?

* Use our tutorial [PING WITH CRONTAB](https://github.com/ALUMLABS/near-docs/ping-with-crontab.md) to automate your NEAR PINGs.