







# ZEOS Validator

How to set up a validator node on a fresh `Ubuntu 22.04`. Your machine should have 64 GB of RAM, 500 GB to 1TB of SSD memory, several high-clock CPU cores and a broadband connection with low latency.

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
#### TODO: increase Go mem buffer size: https://github.com/quic-go/quic-go/wiki/UDP-Receive-Buffer-Size
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
#### TODO: add zeos as peer

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
The generated password should/could look like this: `ub5dcdCQU8s7XsCw`
```
CREATE USER dsp WITH ENCRYPTED PASSWORD '<YOUR-GENERATED-PASSWORD-HERE>';
GRANT ALL PRIVILEGES ON DATABASE dsp to dsp;
CREATE TABLE IF NOT EXISTS blocks (key TEXT NOT NULL UNIQUE, data BYTEA);
CREATE INDEX IF NOT EXISTS blocks_key_text_pattern_ops_idx ON blocks (key text_pattern_ops);
exit
exit
```
`Go` should already be installed from previous nodeos setup.
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

### Zeus SDK
This will install LiquidApps Zeus SDK on your machine. It depends on NodeJS which is installed first.
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












---

How to install Janus:

install:
`sudo apt-get install libavutil-dev libavcodec-dev libavformat-dev`
for post processing of video (https://ourcodeworld.com/articles/read/1198/how-to-join-the-audio-and-video-mjr-from-a-recorded-session-of-janus-gateway-in-ubuntu-18-04)

make sure to `--enable-post-processing` when calling `./configure` when following the instructions on:
./configure --prefix=/opt/janus --enable-post-processing
https://github.com/meetecho/janus-gateway

install libsrtp 2.2.0 as mentioned in the readme (newer versions don't seem to work as of now):

```
wget https://github.com/cisco/libsrtp/archive/v2.2.0.tar.gz
tar xfv v2.2.0.tar.gz
cd libsrtp-2.2.0
./configure --prefix=/usr --enable-openssl
make shared_library && sudo make install
```

create systemd and log rotate file as described here:
https://facsiaginsa.com/janus/install-janus-webrtc-server-on-ubuntu


# Postprocess videos:
janus-pp-rec ./room-1234-user-0001-video.mjr ./video-track.webm
janus-pp-rec ./room-1234-user-0001-audio.mjr ./audio-track.opus
ffmpeg -i ./video-track.webm -i ./audio-track.opus  -c:v copy -c:a opus -strict experimental ./final-video.webm
ffmpeg -i ./final-video.webm -vcodec libx264 -crf 24 final-video.avi

# zeos-validator
