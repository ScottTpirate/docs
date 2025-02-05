---
description: 'Instructions for setting up the rust based relayer, Hermes'
---

# NOTE: THIS IS STILL IN PROGRESS, CHANNELS WILL NEED TO BE SORTED OUT/CHANGED




# Hermes

## Assumptions

We assume that you already have access to Secret, Osmosis, Cosmos, and Sifchain nodes. These can be either local nodes, or you can access them over the network. However, for networked version, you will need to adjust the systemd configuration not to depend on the chains that are run on other servers. And naturally the hermes configuration needs to adjust the addressing of each chain as well.

The given example has all relayed chains run locally, Secret is on standard ports, other chains are configured as follows:

* Osmosis: 36657 and 39090
* Cosmos: 46657 and 49090
* Sifchain: 56657 and 59090

In these instructions, Hermes is installed under /ibc/hermes, adjust the paths according to your setup.

These instructions are based on installation on Debian 10, but should work the same on Debian 11 or recent Ubuntu.

You will need **rust**, **build-essential** and **git** installed to follow these instructions:

```
# rust:
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# build-essential:
sudo apt-get install build-essential -y

# git:
sudo apt install git-all -y
```

## Building Hermes

For preparation, we will create a dedicated user to run Hermes. Following command will also create home directory for the new user.

```text
sudo useradd -m -d /ibc/hermes hermes
```

We will next switch to the hermes user and create a directory where we will compile the relayer software.

```text
sudo sudo -u hermes -s
mkdir /ibc/hermes/source
mkdir /ibc/hermes/bin
cd /ibc/hermes/source
```

Now is time to clone the source repository and build it. Note that we need to checkout the latest release.

```text
git clone https://github.com/informalsystems/ibc-rs.git hermes
cd hermes
git checkout v0.9.0
cargo build --release
cp target/release/hermes ~/bin
cd
```

Next we will check that the newly built hermes version is the correct one:

```text
hermes@demo:~$ bin/hermes version
Nov 04 15:52:48.299  INFO ThreadId(01) using default configuration from '/ibc/hermes/.hermes/config.toml'
hermes 0.8.0
```

## Configuring Hermes

Choose your favourite editor and edit the following configuration template to mach your setup. There are features like telemetry and rest API that you can enable, but they are not necessary, so they are left out from this tutorial.

```text
[global]
strategy = 'packets'
filter = true
log_level = 'info'
clear_packets_interval = 100

#
# Chain configuration Secret
#

[[chains]]
id = 'secret-4'

rpc_addr = 'http://127.0.0.1:26657'
grpc_addr = 'http://127.0.0.1:29090'
websocket_addr = 'ws://127.0.0.1:26657/websocket'

rpc_timeout = '20s'
account_prefix = 'secret'
key_name = 'secret-relayer'
store_prefix = 'ibc'
max_msg_num=15
max_gas = 1000000
gas_price = { price = 0.0125, denom = 'uscrt' }
clock_drift = '5s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3'}

[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-0'], # Cosmos
  ['transfer', 'channel-1'], # Osmosis
  ['transfer', 'channel-2'], # Terra
]


#
# Chain configuration Osmosis
#

[[chains]]
id = 'osmosis-1'

# API access to Osmosis node with indexing
rpc_addr = 'http://127.0.0.1:36657'
grpc_addr = 'http://127.0.0.1:39090'
websocket_addr = 'ws://127.0.0.1:36657/websocket'

rpc_timeout = '20s'
account_prefix = 'osmo'
key_name = 'osmo-relayer'
store_prefix = 'ibc'
max_gas =  1000000
gas_price = { price = 0.000, denom = 'uosmo' }
clock_drift = '5s'
trusting_period = '7days'
trust_threshold = { numerator = '1', denominator = '3' }

[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-88'],
]

#
# Chain configuration Cosmos
#

[[chains]]
id = 'cosmoshub-4'

# API access to Cosmos node with indexing
rpc_addr = 'http://127.0.0.1:46657'
grpc_addr = 'http://127.0.0.1:49090'
websocket_addr = 'ws://127.0.0.1:46657/websocket'

rpc_timeout = '20s'
account_prefix = 'cosmos'
key_name = 'cosmos-relayer'
store_prefix = 'ibc'
max_msg_num=15
max_gas = 1000000
gas_price = { price = 0.0001, denom = 'uatom' }
clock_drift = '5s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }

[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-235'],
]

#
# Chain configuration Sifchain still forthcoming
#

[[chains]]
id = 'sifchain-1'

# API access to Cosmos node with indexing
rpc_addr = 'http://127.0.0.1:56657'
grpc_addr = 'http://127.0.0.1:59090'
websocket_addr = 'ws://127.0.0.1:56657/websocket'

rpc_timeout = '20s'
account_prefix = 'sif'
key_name = 'sif-relayer'
store_prefix = 'ibc'
max_msg_num=15
max_gas = 10000000
gas_price = { price = 0.001, denom = 'rowan' }
clock_drift = '5s'
trusting_period = '14days'
trust_threshold = { numerator = '1', denominator = '3' }

[chains.packet_filter]
policy = 'allow'
list = [
  ['transfer', 'channel-'], # to be determined
]

```

You can validate the configuration with following:

```text
hermes@Demo:~$ bin/hermes -c .hermes/config.toml  config validate
Success: "validation passed successfully"
```

## Setting up wallets

We do this by creating key configuration files that are imported to hermes. Here we go through Secret key setting, other chains are similar.

```text
{
  "name":"secret-relayer",
  "type":"local",
  "address":"secretxxx",
  "pubkey":"{\"@type\":\"/cosmos.crypto.secp256k1.PubKey\",\"key\":\"xxx\"}",
  "mnemonic": "24 words seed"
}
```

Next we will import this key configuration to hermes and shred the used json file. \(Using chain\_id **secret-4**.\)

```text
bin/hermes keys add secret-4 -f ./seed-secret.json
shred -u ./seed-secret.json
```

If you want to make sure the keys got imported, you can check them with following command \(smart thing to run it before shredding the json file\):

```text
bin/hermes keys list secret-4
```

## Testing the setup

Let's do a quick test to see things work properly.

```text
bin/hermes start
```

Once we see things load up correctly and there are no fatal errors, we can break out of hermes with **ctrl-c**.

## Configuring systemd

Now we will setup hermes to be run by systemd, and to start automatically on reboots.

Create the following configuration to **/etc/systemd/system/hermes.service**

```text
[Unit]
Description=Hermes IBC relayer
ConditionPathExists=/ibc/hermes/hermes
After=network.target secret-node.service cosmos.service osmo.service

[Service]
Type=simple
User=hermes
WorkingDirectory=/ibc/hermes
ExecStart=/ibc/hermes/hermes start
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
```

Then we will start hermes with the newly created service and enable it. Note that this step is done from your normal user account that has sudo privileges, so no longer as hermes.

```text
sudo systemctl start hermes.service
sudo systemctl enable hermes.service
```

