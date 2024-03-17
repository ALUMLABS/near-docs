# NEAR PING WITH CRONTAB

__Our tutorials are based on Ubuntu!__

This tutorial requires that you first follow our other two tutorials for securing and running a NEAR validator:

* [SECURE YOUR VALIDATOR](https://github.com/ALUMLABS/near-docs/SECURE-YOUR-VALIDATOR.md)
* [SETUP A VALIDATOR WITH DOCKER](https://github.com/ALUMLABS/near-docs/SETUP-A-VALIDATOR-WITH-DOCKER.md)

If you are stubborn and "did not need our other tutorials" to get you started, just be sure that you executed command `near login` that allows your node to execute transactions on your behalf, found here: https://near-nodes.io/validator/validator-bootcamp#step-2--create-a-wallet

---

## CREATE PING SCRIPT

* Create script: `root@node ~ # nano ~/ping.sh`
* Add this:
    - Substitute `alumlabs.poolv1.near` with your own validator name
    - Substitute `alumlabs.near` with your own wallet name

```bash
#!/bin/bash
# USER SPECIFIC VARS
POOL_ID="alumlabs.poolv1.near"
ACCOUNT_ID="alumlabs.near"
NEAR_ENV="mainnet"
LOG_DIR=/root/near-logs
INFO_LOG="$LOG_DIR/ping.log"
ERROR_LOG="$LOG_DIR/errors.log"
EPOCH_HOLDER="$LOG_DIR/epoch_holder.txt"

# OTHER VARS
export SHELL="/bin/bash"
export LC_CTYPE="C.UTF-8"
export NODE_ENV=$NEAR_ENV
export PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
NOW="$(date)"
NEAR_EXE=$(which near 2>&1 | head -n 1)
JQ_EXE=$(which jq 2>&1 | head -n 1)

# ENSURE LOG_DIR AND LOG FILES EXIST
mkdir -p $LOG_DIR
touch $INFO_LOG
touch $ERROR_LOG
touch $EPOCH_HOLDER

# DEBUG LOCATIONS
#echo "$NOW - NEAR is found at location $NEAR_EXE" >> $INFO_LOG
#echo "$NOW - jq is found at location $JQ_EXE" >> $INFO_LOG

# WE NEED NEAR
if [[ "$NEAR_EXE" = "" ]]; then
    echo -e "\e[1;31m[!] NEAR not found. Please install it first using our tutorial https://github.com/ALUMLABS/near-docs/setup-a-validator-with-docker.md\e[0m"
    echo "$NOW - NEAR not found..." >> $ERROR_LOG
    exit 1
fi

# WE NEED JQ
if [[ "$JQ_EXE" = "" ]]; then
    echo -e "\e[1;31m[!] jq not found. Installing.\e[0m"
    echo "$NOW - JQ not found...I will try to install it now" >> $ERROR_LOG
    apt update
    apt install -y jq
fi

# EPOCH START
epoch_start_height=$(curl -sSf \
    -H 'Content-Type: application/json' \
    -d '{"jsonrpc":"2.0","method":"validators","id":"test","params":[null]}' \
    http://127.0.0.1:3030/ 2>&1)
if [ $? -ne 0 ]; then
    echo -e "\e[1;31m[!] ERROR while fetching epoch via cURL.\e[0m"
    echo "$NOW - Error while fetching epoch_start_height via cURL. Bye!" >> $ERROR_LOG
    exit 1
fi

if [[ "$epoch_start_height" = "" ]]; then
    echo -e "\e[1;31m[!] epoch_start_height is empty...\e[0m"
    echo "$NOW - epoch_start_height is empty. Bye!" >> $ERROR_LOG
    exit 1
fi
epoch_start_height=$(echo "$epoch_start_height" | $JQ_EXE -r .result.epoch_start_height)

# EPOCH DIFF AND PING
prev_epoch=$(cat "$EPOCH_HOLDER")
if [[ "$epoch_start_height" != "$prev_epoch" ]]; then
    echo "$NOW - Sending PING command to the network" >> $INFO_LOG
    echo "$NOW - Via NEAR application: $NEAR_EXE" >> $INFO_LOG
    execute=`NEAR_ENV=$NEAR_ENV $NEAR_EXE call $POOL_ID ping '{}' --accountId $ACCOUNT_ID >> $INFO_LOG 2>&1`
    if [ $? -ne 0 ]; then
        echo "$NOW - Error while running PING command" >> $ERROR_LOG
        echo "$execute" >> $ERROR_LOG
    else
        echo $epoch_start_height > $EPOCH_HOLDER
    fi
else
    echo "$NOW - No epoch change ($epoch_start_height vs $prev_epoch)" >> $INFO_LOG
fi
```

* Run the script once to test it out: `root@node ~ # bash /root/ping.sh`

---

## CONFIGURE LOG ROTATION

The PING script generates log files. To not fill up your disk with old log files, we rotate them.

* Create file: `root@node ~ # nano /etc/logrotate.d/near-ping`
* Add this:

```
/root/near-logs/ping.log {
        su root root
        daily
        missingok
        rotate 14
        compress
        notifempty
        create 0775 root root
}
/root/near-logs/errors.log {
        su root root
        daily
        missingok
        rotate 14
        compress
        notifempty
        create 0775 root root
}
```

* After running the PING script once, run `logrotate --force /etc/logrotate.d/near-ping` to test the log rotation. It should create `*.gz` files inside the log directory and only save up to 14 version.

---

## ACTIVATE CRONTAB

Automate everything with crontab!

* Edit crontab: `root@node ~ # crontab -e`
    - Use option `nano` if you did not choose your default editor yet.
* Add this: 

```bash
# PING NEAR
@hourly /bin/bash /root/ping.sh >/dev/null 2>&1
```

Check if you see PING commands appear here after an hour: https://nearblocks.io/address/alumlabs.poolv1.near

* *(Substitute `alumlabs` with your own validator name)*