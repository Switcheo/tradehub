# Getting started

## Requirements

You will need a Linux, Ubuntu 18 compute instance for running your Switcheo TradeHub node.

Switcheo TradeHub installation only works on Ubuntu 18 at the moment, other Linux flavors will be available in the near future.

Both a VPS or bare-metal machine will work, but [security considerations](#Validator-Security) should be taken into account.

Additionally, running a full node requires Redis, Postgres and Nginx for storing / serving off-chain data and APIs, which will automatically be installed and configured in the following script.

Therefore, nodes should be run as a dedicated instance to prevent configuration conflicts. For development or testing, a docker container can be used and will be made available later on.

Here are the minimum system requirements for a validator node:

**Mainnet**: 8GB RAM, 4 CPUs, 5TB Storage

The 5TB requirement is high due to our 1 second block times and is an estimate based on 1 year of high trading volume. Pruning and compression solutions for old block data to reduce storage requirements is currently our top-most priority. Validators may join the network with 1TB and expand their storage later if required.

Mainnet validators are strongly recommended to read about [sentry nodes](#sentry-nodes-ddos-protection) to help in DDOS protection.

**Testnet**: 2GB RAM, 2 CPUs, 200GB Storage

## Download a `switcheoctl` release

The following package contains the Switcheo TradeHub client, as well as various tools
that may be useful in running a validator or seed node: [https://github.com/Switcheo/tradehub/releases](https://github.com/Switcheo/tradehub/releases)

You may use the following command to download and unzip the release:

### Testnet Release

```bash
curl -L https://github.com/Switcheo/tradehub/releases/download/v1.8.6/install-testnet.tar.gz | tar -xz
```

### Mainnet Release

```bash
curl -L https://github.com/Switcheo/tradehub/releases/download/v1.9.0/install-mainnet.tar.gz | tar -xz
```

## Install `switcheoctl`

`switcheoctl` allows you to install and control your Switcheo TradeHub node easily.

Install it with the following command:

### Testnet Install

```bash
cd install-testnet && sudo ./install.sh && cd - && rm -rf install-testnet
```

### Mainnet Install

```bash
cd install-mainnet && sudo ./install.sh && cd - && rm -rf install-mainnet
```

### Sentry / public node setup

If you are setting up a sentry node, this should be done before setting up your validator node. If you are setting up a public seed / peer node, this setup step will work as well.

1. Update the node config file that can be found at `/etc/switcheoctl/config/config.json`:
   1. `sudo nano /etc/switcheoctl/config/config.json`
   2. Choose a unique monikier that identifies you well and replace `hikaru` with it.
   3. You should allow your node to connect to other public nodes by configuring your
   `persistent_peers`, we have provided the default as Switcheo's sentry node. This should be given as a comma separated list of nodes in `<node_id>@<ip_address>:26656` format.
   4. It is important to manage your persistent peers. More on that [here](#managing-p2p-connections).
2. Install with: `switcheoctl install-node`

### Validator node setup

1. Update the node config file that can be found at `/etc/switcheoctl/config/config.json`:
   1. `sudo nano /etc/switcheoctl/config/config.json`
   2. Choose a unique monikier that identifies you well and replace `hikaru` with it.
   3. If you are using a sentry node, edit `persistent_peers` with the connection info for your sentry nodes. This should be given as a comma separated list of nodes in `<node_id>@<ip_address>:26656` format.

      For example: `4ff8a406fc16e8b4bc0db59556295f8618fda1d7@52.1.1.1:26656,58995fb71258f3fbccbb79a28164895ffba70d74@52.1.1.2:26656`

      You can find your `node_id` by curl-ing: `http://<ip_address>:26657/status`.

   4. Set `pex` to `false` if you have sentry nodes. If you do not have sentry nodes, it is important to manage your persistent peers. More on that [here](#managing-p2p-connections).
   5. You may update other details for your validator later on.
2. Install with: `switcheoctl install-validator`
3. Create the required wallets for running a validator node. **Store the generated mnemonics in a safe, offline location!**

     ```bash
     switcheocli keys add val --keyring-backend file
     # You will need to choose a secure password when adding the first wallet.
     # All other wallets in the keyring must use the same password.
     # Store the generated mnemonics for each wallet as a backup!
     switcheocli keys add oraclewallet --keyring-backend file
     switcheocli keys add liquidator --keyring-backend file
     ```

    Each wallet serves a different role:
    - `val`: Your validator operator wallet.
    - `oraclewallet`: Oracle subaccount wallet - validators are oracles at genesis and will need to submit oracle result txns when trading begins. In future oracles will be incentivized through a separate incentive model.
    - `liquidator`: Liquidator subaccount wallet - validators are liquidators at genesis and will need to submit liquidation txns when trading begins. In future liquidators will be incentivized through a separate incentive model.

    The oracle and liquidator services require public HTTP access to run. If your validator machine does not have such access, you should create those wallets on another machine running a public node (same as sentry node configuraton). These wallets must be bound as subaccounts to the main validator operator wallet through the `subaccounts` command / transaction, more info [here](#link-your-wallets-through-subaccounts). **If you do so, you should also create the `val` wallet on a separate machine, and ensure that you do not set any wallet password configuration on supervisord or switcheoctl (leave empty went prompted).**

    The oracle and liquidator services can be ran separately with the `switcheod oracle` and `switcheod liquidator` commands. However, these will do nothing until trading begins.

4. Send SWTH to all wallets for self-staking and paying network fees. You can deposit NEP-5 SWTH into Switcheo TradeHub and then transfer SWTH from another wallet through the following command:

    `switcheocli tx bank send --from <from_name> --keyring-backend file -y --fees 100000000swth -b block val <to_address> <amountInSatoshis>swth`

    Example to send 1000 SWTH from `mywallet` to `swth1haze3..n5e`:

    `switcheocli tx bank send --from mywallet --keyring-backend file -y --fees 100000000swth -b block val swth1haze3ah2pwdhgstw9v5fcfphqccp359xgrgn5e 100000000000swth`

    If you are setting up a testnet node, contact one of a Switcheo support member for some fake SWTH!

## Running your node

Start your node using `switcheoctl`:

If you are starting a sentry node, non-validator node, or a validator node without public internet access, start it using:

`switcheoctl start -n`

If you are starting a validator node together with all required services, start it using:

`switcheoctl start`

If you are running a validator node behind a sentry node, update your sentry node configuration to not gossip your validator node's IP:

```bash
# In the validator node(s):

# Get your node ID(s):
$ curl http://localhost:26659/status

#  "jsonrpc": "2.0",
#  "id": -1,
#  "result": {
#    "node_info": {
#        ...
#      },
#      "id": "f09e200a655ce63e49b3710653258a674730e036",
#      ...

# node_id is `f09e200a655ce63e49b3710653258a674730e036`

# In the sentry node(s):

# Edit your node config:
$ vi ~/.switcheod/config/config.toml

# Comma separated list of peer IDs in `<node_id` format to keep private (will not be gossiped to other peers), e.g:
private_peer_ids = "f09e200a655ce63e49b3710653258a674730e036"

# Restart your sentry node:
$ switcheoctl restart -n
```

### Managing p2p connections

Your sentry node should define a list of persistent peers that you know will always be online. After installing your node, you should update persistent_peers in `~/.switcheod/config/config.toml`, and restart your node. Ensure that your sentry node's `pex` is set to true in `~/.switcheod/config/config.toml`, otherwise only the persistent_peers list is available for connection.

You may add Switcheo's sentry nodes, but it would be better to add other sentry nodes as well: `"d363e17a3d4c7649e5c59bcd33176a476433108c@54.179.34.89:26656,ca1189045e84d2be5db0a1ed326ce7cd56015f11@54.255.5.46:26656,d233cd97f8c81b7705ea16b962a4101063875151@175.41.151.35:26656"`

### Installing your node to a data directory

By default, the Switcheo TradeHub node is configured to run from the home directory of the installing user.

To run it from a different data directory, simply copy the node files into your data directory and symlink the copied files and folders.

For example to run from `/data`:

```bash
switcheoctl stop

# Change the destination directory here
target=/data

mv ~/.switcheo* $target && cd $target

declare -a dirs=(
    "switcheocli"
    "switcheod"
)

for dir in ${dirs[@]}; do mv .$dir $dir && ln -s $target/$dir ~/.$dir; done

unset dirs target
switcheoctl start
```

Ensure file and folder permissions remain unchanged.

## Stake as a validator

**:exclamation: WARNING :exclamation:**

**You should check that the node has caught up to latest block by running `switcheocli status` before continuing! If your node is still syncing (`sync_info.catching_up: true`), it will not be able to validate new blocks and you will end up getting slashed / jailed.**

Ensure your node is healthy to avoid getting your stake slashed. You can find info on your node at:

  `curl localhost:26659/abci_info`

You can check that your wallets have sufficient SWTH after starting through:

   `curl http://localhost:5001/auth/accounts/<address>`

You may need some time for your node to sync to see updated information.

Promote your node to validator with this command:

```bash
# alias command if wallets are on validator machine:
$ switcheoctl create-validator -a <amountToStakeInSatoshis>

# OR, if creating a validator node without any wallets:

# get node public key on validator machine
# testnet: switcheod node tendermint show-validator
$ switcheod tendermint show-validator
swthvalconspub1zcjduepqqsuvl3xj58qmfv49je....

# on operator machine
# NOTE: default commision rates are used in this command, but you should change them as needed
# --commission-max-change-rate 0.010000000000000000: Max increase of 1% in commission rate every 24 hours
# --commission-max-rate 0.200000000000000000: Max 20% commission rate
# --commission-rate 0.100000000000000000: Start with 10% commission rate
# --min-self-delegation 1: Minimum number of SWTH tokens that must be delegated
switcheocli tx staking create-validator --amount <amountToStakeInSatoshis>swth --moniker <yourmoniker> --pubkey <pubkey> --commission-max-change-rate 0.010000000000000000 --commission-max-rate 0.200000000000000000 --commission-rate 0.100000000000000000 --min-self-delegation 1 --fees 100000000swth -b block --from val --keyring-backend file -y
```

You can check that is is successful by getting your validator operator address and looking up the chain:

```bash
switcheocli keys show val --keyring-backend file --bech val -a

curl http://localhost:5001/staking/validators/<val_address>
```

You may update your validator information with:

```bash
switcheocli tx staking edit-validator
--moniker="choose a name" \
--website="https://yourwebsite.example.com" \
--security-contact="security@example.com" \
--identity=<your_keybase_hash> \
--details="Choose a good description for yourself / your company." \
--commission-rate="0.10" \
--from=val \
--fees 100000000swth \
--chain-id=<chain_id> \
--keyring-backend file \
--yes
```

The validator details are similar to that of Cosmos Hub and can be found [here](https://hub.cosmos.network/master/validators/validator-setup.html#edit-validator-description).

## Link your wallets through subaccounts

The `oraclewallet` should be linked to your `val` wallet as a subaccount, this will allow the `oraclewallet` to send oracle votes on behalf of your `val` wallet.
Before linking the wallets, it should be ensured that both the `val` and `oraclewallet` addresses have sufficient funds to pay fees.

To link the `oraclewallet` as a subaccount of your `val` wallet, you can use the following cli commands:

  ```bash
  switcheocli tx subaccount create-sub-account --from val --keyring-backend file -y --fees 100000000swth -b block val <oraclewallet-swth-address> <val-swth-address>
  switcheocli tx subaccount activate-sub-account --from oraclewallet --keyring-backend file -y --fees 100000000swth -b block oraclewallet <oraclewallet-swth-address> <val-swth-address>
  ```

You should run `switcheoctl restart` after linking your wallets, this will allow the oracle service to start correctly.

## Upgrading your node

When upgrading a minor binary version (e.g. 1.8.0 to 1.8.1), there should be no changes in consensus or chain state. In this case, we can simply patch the node and CLI binaries - `switcheod` and `switcheocli` by hot replacing them into v1.8.0 folder, which is where cosmovisor/current is pointing to. This should only be done when your chain have been fully caught up.

If you have a sentry node, upgrade sentry node first then upgrade validator node.

You would need to perform the following steps fast to prevent getting slashed, if you are a validator node.

```bash
# download switcheoctl, pick the version to upgrade from https://github.com/Switcheo/tradehub/releases
curl -L https://github.com/Switcheo/tradehub/releases/download/vx.x.x/install-<mainnet|testnet>.tar.gz | tar -xz

# install switcheoctl
cd install-<mainnet|testnet> && sudo ./install.sh && cd - && rm -rf install-<mainnet|testnet>

# make sure you have backed up v1.8.0 (usually latest /upgrades/vx.x.0)
cp -r ~/.switcheod/cosmovisor/upgrades/v1.8.0 ~/.switcheod/cosmovisor/upgrades/v1.8.0.bak

# stop services
switcheoctl stop
# replace switcheod in v1.8.0, cosmovisor/current is pointing to this
sudo cp /etc/switcheoctl/bin/switcheod ~/.switcheod/cosmovisor/upgrades/v1.8.0/bin
# replace switcheocli, cosmovisor/current is pointing to this
sudo cp /etc/switcheoctl/bin/switcheocli ~/.switcheod/cosmovisor/upgrades/v1.8.0/bin

# start validator or
switcheoctl start
# start sentry
switcheoctl start -n

# check that chain is progressing, there should be no errors
tail -f ~/.switcheod/logs/*

# check version is patched correctly
curl -s localhost:1318/node_info | jq -r '.application_version.version'
```

---

## Validator Security

Each validator candidate is encouraged to run its operations independently, as diverse setups increase the resilience of the network. The following guide is adapted from the Cosmos Hub security page.

### Key Management

It is critical that an attacker cannot steal your validator key. If this is possible, it puts the entire stake delegated to the compromised validator at risk. Hardware security modules are a way to mitigate this risk.

HSM modules must support ed25519 signatures for the hub. Please see the [KMS guide](./KMS.md) for how you may use Tendermint KMS to interface with either the Ledger or YubiHMS2 hardware modules.

It is also important that only one instance of each validator node is active at any time. If two copies of a node is ran with the same validator key, the validator will likely sign the same block and be slashed. As such, any redundant setup where there is a backup node on standby must be configured with extreme care.

### Sentry Nodes (DDOS Protection)

Validators are responsible for ensuring that the network can sustain denial of service attacks.

One recommended way to mitigate these risks is for validators to carefully structure their network topology in a so-called [sentry node architecture](https://forum.cosmos.network/t/sentry-node-architecture-overview/454). Validators without sentry nodes are **very susceptible to spam** which would cause them to go down.

Validator nodes should only connect to full-nodes they trust because they operate them themselves or are run by other validators they know socially. A validator node will typically run in a data center. Most data centers provide direct links the networks of major cloud providers. The validator can use those links to connect to sentry nodes in the cloud. This shifts the burden of denial-of-service from the validator's node directly to its sentry nodes, and may require new sentry nodes be spun up or activated to mitigate attacks on existing ones.

Sentry nodes can be quickly spun up or change their IP addresses. Because the links to the sentry nodes are in private IP space, an internet based attacked cannot disturb them directly. This will ensure validator block proposals and votes always make it to the rest of the network.

To setup your sentry node architecture you can follow the instructions below:

Validators nodes should edit their config.toml:

```bash
# Comma separated list of nodes to keep persistent connections to
# Do not add private peers to this list if you don't want them advertised
persistent_peers =[list of sentry nodes]

# Set true to enable the peer-exchange reactor
pex = false
```

Sentry Nodes should edit their config.toml:

```bash
# Comma separated list of peer IDs to keep private (will not be gossiped to other peers)
# Example ID: 3e16af0cead27979e1fc3dac57d03df3c7a77acc@3.87.179.235:26656

private_peer_ids = "node_ids_of_private_peers"
```

Validator nodes should only open the P2P port (26656) to their sentry nodes ip and rely on your sentry nodes for serving other requests. See the FAQ below for more information.

---

## Increase maximum number of open file descriptors

If you encounter "too many open files" in your logs, you need to increase your open file descriptors.
```bash
# show current limit
cat /etc/systemd/system/switcheod.service | grep LimitNOFILE
```

Increase LimitNOFILE
```bash
sudo vi /etc/systemd/system/switcheod.service
```

Finally, restart your node
```bash
switcheoctl restart # for validator node
switcheoctl restart -n # for sentry node
```

---

## FAQ

### Ports used

#### Validator Node

- 26656 - P2P

#### Sentry Node

- 5001 - Nginx reverse proxy to Cosmos SDK API and REST API
- 1318 - Cosmos SDK API
- 1317 - Nginx reverse proxy to Cosmos SDK API
- 5000 - REST WS
- 5002 - REST API
- 26656 - P2P
- 26659 - Tendermint API
- 26657 - Nginx reverse proxy to Tendermint API

### Inbound traffic ports

#### Validator Node

The following ports should be open to allow for p2p traffic between nodes:

- 26656 - P2P (if you have sentry nodes, only open this port to their ip)

#### Sentry Node

The following ports should be open to allow inbound query traffic:

- 5001 - Nginx reverse proxy to Cosmos SDK API and REST API
- 1317 - Nginx reverse proxy to Cosmos SDK API
- 5000 - REST WS
- 5002 - REST API (optional when using nginx proxy)
- 26656 - P2P
- 26657 - Nginx reverse proxy to Tendermint API

### Chain ID

The chain IDs are already configured appropriately if you are using the correct package.

- Testnet: `switcheochain`
- Mainnet: `switcheo-tradehub-1`

### Unjail your jailed validator node

1. Ensure your node is not still syncing before unjailing or you will be slashed again!

2. Send a unjail txn:

    `switcheocli tx slashing unjail --from val --keyring-backend file -y --fees 100000000swth -b block`

### Unstake your self-staked tokens

1. Get your validator address:

    `switcheocli keys show val --keyring-backend file --bech val -a`

2. Send an unstake txn:

    `switcheocli tx staking unbond <val_address> 100000000000swth --from val --fees 100000000swth -y -b block --keyring-backend file`

### Stake additional tokens

1. Get your validator address:

    `switcheocli keys show val --keyring-backend file --bech val -a`

2. Send a stake txn:

    `switcheocli tx staking delegate <val_address> 100000000000swth --from val --fees 100000000swth -y -b block --keyring-backend file`

### Enable switcheod services to auto start after boot

1. Check status of switcheod.service is-enabled

   `sudo systemctl is-enabled switcheod.service`

2. Enable switcheod.service

    `sudo systemctl enable switcheod.service`

## Debugging

Logs for all services / processes under Switcheo TradeHub are written to the .switcheod/logs folder.

You can tail all logs for debugging with:

`tail -f ~/.switcheod/logs/*`

To tail a specific service's log:

`tail -f ~/.switcheod/logs/node_start.log`
