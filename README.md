# ZEOS Validator

How to set up a validator node on a fresh `Ubuntu 22.04`. Your machine should have 64 GB of RAM, 500 GB to 1TB of SSD memory, a fast CPU with several cores and a broadband internet connection with low latency.

## Prerequisites

- make sure current user is in sudoers: `sudo usermod -aG sudo user` where you replace `user` with the current user `echo $USER`.
- consider using 'screen' session: `sudo apt install screen` you can then reconnect using `screen -rd` in case your ssh session crashed.
- create EOS blockchain account for your DSP: Head over to [bloks.io](https://bloks.io/wallet/create-account) and create a new EOS keypair and account.

## Setup DSP

The instructions here should work on a fresh install of Ubuntu 22.04.

### nodeos
The following instructions will set up nodeos as `SHIP` node on your machine without history solution.
```
cd
wget https://github.com/AntelopeIO/leap/releases/download/v3.2.1/leap_3.2.1-ubuntu22.04_amd64.deb
sudo dpkg -i leap_3.2.1-ubuntu22.04_amd64.deb
# verify installation:
nodeos --full-version
# should print: v3.1.2-0b64f879e3ebe2e4df09d2e62f1fc164cc1125d1
```
Get latest snapshot from [snapshots.eosnation.io](https://snapshots.eosnation.io/)
Replace filename with latest snapshot of EOS v6:
```
wget https://pub.store.eosnation.io/eos-snapshots/snapshot-2023-03-07-16-eos-v6-0298042560.bin.zst
unzstd snapshot-2023-03-07-16-eos-v6-0298042560.bin.zst
mkdir $HOME/.local/share/eosio/nodeos/data/snapshots -p
mkdir $HOME/.local/share/eosio/nodeos/config -p
mv snapshot-2023-03-07-16-eos-v6-0298042560.bin $HOME/.local/share/eosio/nodeos/data/snapshots/boot.bin
# get active peers
wget https://validate.eosnation.io/eos/reports/config.txt
mv config.txt $HOME/.local/share/eosio/nodeos/config/peers.txt
# get genesis json
wget https://raw.githubusercontent.com/CryptoLions/EOS-MainNet/master/genesis.json
mv genesis.json $HOME/.local/share/eosio/nodeos/config/
```
Configure nodeos:
```
cd $HOME/.local/share/eosio/nodeos/config
cat <<EOF >> config.ini
```
Copy and paste the following 43 lines and press ENTER:
```
agent-name = "DSP"
# for firehose comment in the below plugin
# deep-mind = true
http-server-address = 0.0.0.0:8888
p2p-listen-endpoint = 0.0.0.0:9876
blocks-dir = "blocks"
abi-serializer-max-time-ms = 3000
max-transaction-time = 150000
wasm-runtime = eos-vm
eos-vm-oc-enable = true
contracts-console = true
p2p-max-nodes-per-host = 1
allowed-connection = any
max-clients = 100
sync-fetch-span = 500
connection-cleanup-period = 30
http-validate-host = false
access-control-allow-origin = *
access-control-allow-headers = *
access-control-allow-credentials = false
verbose-http-errors = true
http-threads=4
net-threads=4
chain-threads=4
eos-vm-oc-compile-threads=2
trace-history-debug-mode = true
trace-history = true
http-max-response-time-ms = 500
transaction-retry-max-storage-size-gb = 4
transaction-finality-status-max-storage-size-gb = 4
disable-subjective-billing = true
block-log-retain-blocks = 1000
state-history-log-retain-blocks = 1000
transaction-retry-max-expiration-sec = 300
p2p-dedup-cache-expire-time-sec = 3
transaction-retry-interval-sec=6
plugin = eosio::producer_plugin
plugin = eosio::chain_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::net_plugin
plugin = eosio::state_history_plugin
state-history-endpoint = 0.0.0.0:8887
chain-state-db-size-mb = 32768
EOF
```
Append peers to config.ini:
```
cat peers.txt >> config.ini
```
Start nodeos for the first time using the downloaded snapshot:
```
nodeos --disable-replay-opts --snapshot $HOME/.local/share/eosio/nodeos/data/snapshots/boot.bin --delete-state-history --delete-all-blocks
```
You will know that the node is fully synced once you see blocks being produced every half second at the head block. You can match the block number you are seeing in the nodeos logs to what bloks.io is indicating as the head block on the chain you are syncing (mainnet, Kylin etc). Once you have confirmed that it is synced press CTRL+C once, wait for the node to shutdown and proceed to the next step.
Create daemon config:
```
cat <<EOF | sudo tee /lib/systemd/system/nodeos.service
```
Copy and paste the following 9 lines:
```
[Unit]
Description=nodeos
After=network.target
[Service]
User=$USER
ExecStart=/usr/bin/nodeos --disable-replay-opts
[Install]
WantedBy=multi-user.target
EOF
```
Start daemon:
```
sudo systemctl start nodeos
sudo systemctl enable nodeos
# check if nodeos is running
systemctl status nodeos
```

### IPFS
This will install `go-ipfs` on your machine and configure an IPFS daemon. 
```
cd
sudo apt-get update
sudo apt-get install golang-go -y 
wget http://dist.ipfs.io/go-ipfs/v0.4.22/go-ipfs_v0.4.22_linux-amd64.tar.gz
tar xvfz go-ipfs_v0.4.22_linux-amd64.tar.gz
sudo mv go-ipfs/ipfs /usr/local/bin/ipfs
```
Increase Go memory buffer size:
```
sudo sysctl -w net.core.rmem_max=2500000
```
Configure IPFS daemon:
```
sudo su -
ipfs init
ipfs config Addresses.API /ip4/0.0.0.0/tcp/5001
ipfs config Addresses.Gateway /ip4/0.0.0.0/tcp/8080
cat <<EOF > /lib/systemd/system/ipfs.service
```
Copy and paste the following 9 lines and press ENTER:
```
[Unit]
Description=IPFS daemon
After=network.target
[Service]
ExecStart=/usr/local/bin/ipfs daemon
Restart=always
[Install]
WantedBy=multi-user.target
EOF
```
Start daemon:
```
systemctl start ipfs
systemctl enable ipfs
exit
# check if ipfs is running
systemctl status ipfs
```
#### TODO: add zeos node as a peer?

### PostgreSQL
This will install PostgreSQL on your machine and configure a daemon.
```
cd
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list
sudo apt update
sudo apt -y install postgresql-12 postgresql-client-12
sudo su - postgres
psql
CREATE DATABASE dsp;
```
Generate a password, for instance here: [passwordsgenerator.net](https://passwordsgenerator.net/) (disable Symbols)
The generated password could look like this: `ub5dcdCQU8s7XsCw`
```
CREATE USER dsp WITH ENCRYPTED PASSWORD '<YOUR-GENERATED-PASSWORD-HERE>';
GRANT ALL PRIVILEGES ON DATABASE dsp to dsp;
CREATE TABLE IF NOT EXISTS blocks (key TEXT NOT NULL UNIQUE, data BYTEA);
CREATE INDEX IF NOT EXISTS blocks_key_text_pattern_ops_idx ON blocks (key text_pattern_ops);
exit
exit
```
`Go` should already be installed from previous nodeos setup. Execute:
```
go install github.com/alanshaw/ipfs-ds-postgres@latest
```
Configure postgres daemon:
```
sudo su -
cat <<EOF > /lib/systemd/system/ipfs-ds-postgres.service
```
Copy and paste the following 17 lines:
```
[Unit]
Description=IPFS PostgreSQL alternative backend
After=network.target
[Service]
User=root
# update working directory
# echo $(readlink -f `which setup-dsp` | xargs dirname)/../ipfs-ds-postgres/services/ipfs-ds-postgres
WorkingDirectory=/usr/lib/node_modules/@liquidapps/dsp/zeus_boxes/ipfs-ds-postgres/services/ipfs-ds-postgres
Environment="DATABASE_URL=postgres://dsp:<YOUR-GENERATED-PASSWORD-HERE>@localhost:5432/dsp"
# determine go install path with below command, if different than below, update
# which go
ExecStart=/usr/bin/go run main.go
KillMode=process
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```
Start daemon:
```
systemctl start ipfs-ds-postgres
systemctl enable ipfs-ds-postgres
exit
# check if postgres is running
systemctl status ipfs-ds-postgres 
```

### Zeus SDK (DSP node)
This will install LiquidApps Zeus SDK on your machine and run a services node. It depends on NodeJS which is installed first.
```
cd
sudo apt-get update
sudo apt-get install -y gcc g++ make
curl -sL https://deb.nodesource.com/setup_18.x -o nodesource_setup.sh
sudo bash nodesource_setup.sh
sudo apt-get install -y nodejs
sudo apt install -y make cmake build-essential python3 git node-typescript
sudo su -
npm install -g pm2
npm install -g @liquidapps/dsp --unsafe-perm=true
mkdir ~/.dsp
cp $(readlink -f `which setup-dsp` | xargs dirname)/sample-config.toml ~/.dsp/config.toml
nano ~/.dsp/config.toml
```
Make the following changes:
- line 5: change `<DSP ACCOUNT>` to the EOS account name of your DSP
- line 7: change `<DSP PRIVATE KEY>` to the private key of the active permission of your DSP account
Scroll down to section `[database]` and change:
- line 2: change `postgres://user:pass@example.com:5432/dbname` to `postgres://dsp:<YOUR-GENERATED-PASSWORD-HERE>@localhost:5432/dsp`
Use the same password as generated for the Postgres installation!
For example, the result could look like this: `postgres://dsp:ub5dcdCQU8s7XsCw@localhost:5432/dsp`
If you're done press `ctrl+o` and ENTER to save the file. Press `ctrl+x` to exit.
Next start the dsp:
```
cd $(readlink -f `which setup-dsp` | xargs dirname)
setup-dsp
exit
```
Check if your DSP is up and running by going to the following URL in a web browser: `http://<IP-ADDRESS-OF-YOUR-NODE-HERE>:3115/v1/dsp/version`
It should show something like: `2.0.8213-latest`

## Register DSP service packages
A ZEOS Validator should offer the following services:
- Oracle
- LiquidStorage
- VRAM
Install the zeus command line interface:
```
cd
sudo su -
npm install -g @liquidapps/zeus-cmd
```
Change `line 42` in the following file:
```
nano $(readlink -f `which setup-dsp` | xargs dirname)/../seed-eos/tools/eos/networks.js
```
from:
```
        host: 'bp.cryptolions.io',
```
to:
```
        host: 'localhost',
```
to make zeus use your own node to execute transactions (`bp.cryptolions.io` is down).
Now let's create the for the ZEOS Shielded Protocol required service packages.

### Oracle
```
export PACKAGE=oracle
export DSP_ACCOUNT=<YOUR-DSP-ACCOUNT-HERE>
# active key to sign package creation trx
export DSP_PRIVATE_KEY=<YOUR-DSP-PRIVATE-KEY-HERE>
# customizable and unique name for your package
export PACKAGE_ID=package1
export EOS_CHAIN=mainnet
# the minimum stake quantity is the amount of DAPP and/or DAPPHDL that must be staked to meet the package's threshold for use
export MIN_STAKE_QUANTITY="10.0000"
# package period is in seconds, so 86400 = 1 day, 3600 = 1 hour
export PACKAGE_PERIOD=3600
# the time to unstake is the greater of the package period remaining and the minimum unstake period, which is also in seconds
export MIN_UNSTAKE_PERIOD=3600
# QUOTA is the measurement for total actions allowed within the package period to be processed by the DSP.  1.0000 QUOTA = 10,000 actions. 0.0001 QUOTA = 1 action
export QUOTA="1.0000"
export DSP_ENDPOINT=http://89.117.58.26:3115/
# package json uri is the link to your package's information, this is customizable without a required syntax
export PACKAGE_JSON_URI=http://89.117.58.26:3115/package1.dsp-package.json
# The annual inflation rate of the DAPP token may be tuned by the community. This is done by DSP's specifying a desired inflation rate during package registration. All existing packages default to the original annual inflation rate of 2.71%
export ANNUAL_INFLATION=2.71

cd $(readlink -f `which setup-dsp` | xargs dirname)/../..
zeus register dapp-service-provider-package \
    $PACKAGE $DSP_ACCOUNT $PACKAGE_ID \
    --key $DSP_PRIVATE_KEY \
    --min-stake-quantity $MIN_STAKE_QUANTITY \
    --package-period $PACKAGE_PERIOD \
    --quota $QUOTA \
    --network $EOS_CHAIN \
    --api-endpoint $DSP_ENDPOINT \
    --package-json-uri $PACKAGE_JSON_URI \
    --min-unstake-period $MIN_UNSTAKE_PERIOD \
    --inflation $ANNUAL_INFLATION
```

### IPFS
Only need to change the package type and id and then submit again:
```
export PACKAGE=ipfs
export PACKAGE_ID=package2

zeus register dapp-service-provider-package \
    $PACKAGE $DSP_ACCOUNT $PACKAGE_ID \
    --key $DSP_PRIVATE_KEY \
    --min-stake-quantity $MIN_STAKE_QUANTITY \
    --package-period $PACKAGE_PERIOD \
    --quota $QUOTA \
    --network $EOS_CHAIN \
    --api-endpoint $DSP_ENDPOINT \
    --package-json-uri $PACKAGE_JSON_URI \
    --min-unstake-period $MIN_UNSTAKE_PERIOD \
    --inflation $ANNUAL_INFLATION
```

### Storage
Only need to change the package type and id and then submit again:
```
export PACKAGE=storage
export PACKAGE_ID=package3

zeus register dapp-service-provider-package \
    $PACKAGE $DSP_ACCOUNT $PACKAGE_ID \
    --key $DSP_PRIVATE_KEY \
    --min-stake-quantity $MIN_STAKE_QUANTITY \
    --package-period $PACKAGE_PERIOD \
    --quota $QUOTA \
    --network $EOS_CHAIN \
    --api-endpoint $DSP_ENDPOINT \
    --package-json-uri $PACKAGE_JSON_URI \
    --min-unstake-period $MIN_UNSTAKE_PERIOD \
    --inflation $ANNUAL_INFLATION
exit
```

### Enable Packages
Head over to [bloks.io](https://bloks.io/account/dappservices?loadContract=true&tab=Actions&account=dappservices&scope=dappservices&limit=100&table=package&action=enablepkg) and enable your packages. Import your DSP private key into Anchor and connect your account on bloks. Make sure to have enough CPU, RAM and NET available on your DSP account. Then execute the `enablepkg` action with the following parameters:
```
provider: <YOUR-DSP-ACCOUNT-HERE>
package_id: package1
service: oracleservic
```
Execute the same action for the other two packages:
```
provider: <YOUR-DSP-ACCOUNT-HERE>
package_id: package2
service: ipfsservice1
```
and:
```
provider: <YOUR-DSP-ACCOUNT-HERE>
package_id: package3
service: liquidstorag
```

Your packages are now registered and enabled and can be subscribed to by `thezeostoken` to increase decentralization.
