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

Mainnet validators should also read about [sentry nodes](#sentry-nodes-ddos-protection) to help in DDOS protection.

**Testnet**: 2GB RAM, 2 CPUs, 200GB Storage

## Download a `switcheoctl` release

The following package contains the Switcheo TradeHub client, as well as various tools
that may be useful in running a validator or seed node: [https://github.com/Switcheo/tradehub/releases](https://github.com/Switcheo/tradehub/releases)

You may use the following command to download and unzip the release:

### Testnet Release

```bash
curl -L https://github.com/Switcheo/tradehub/releases/download/v1.2.2/install-testnet.tar.gz | tar -xz
```

### Mainnet Release

```bash
curl -L https://github.com/Switcheo/tradehub/releases/download/v1.2.2/install-mainnet.tar.gz | tar -xz
```

## Install `switcheoctl`

`switcheoctl` allows you to install and control your Switcheo TradeHub node easily.

Install it with the following command:

### Testnet Install

```bash
cd install-testnet && sudo ./install-testnet.sh && cd - && rm -rf install-testnet
```

### Mainnet Install

```bash
cd install-mainnet && sudo ./install-mainnet.sh && cd - && rm -rf install-mainnet
```

### Sentry / public node setup

If you are setting up a sentry node, this should be done before setting up your validator node. If you are setting up a public seed / peer node, this setup step will work as well.

1. Update the node config file that can be found at `/etc/switcheoctl/config/config.json`:
   1. `sudo nano /etc/switcheoctl/config/config.json`
   2. Choose a unique monikier that identifies you well and replace `hikaru` with it.
   3. You should allow your node to connect to one other public node by configuring your
   `persistent_peers`, we have provided the default as Switcheo's sentry node.
2. Install with: `switcheoctl install-node`

### Validator node setup

1. Update the node config file that can be found at `/etc/switcheoctl/config/config.json`:
   1. `sudo nano /etc/switcheoctl/config/config.json`
   2. Choose a unique monikier that identifies you well and replace `hikaru` with it.
   3. If you are using a sentry node, edit `persistent_peers` with the connection info for your sentry nodes. This should be given as a comma separated list of nodes in `<node_id>@<ip_address>:26656` format.

      For example: `4ff8a406fc16e8b4bc0db59556295f8618fda1d7@52.1.1.1:26656,58995fb71258f3fbccbb79a28164895ffba70d74@52.1.1.2:26656`

      You can find your `node_id` by curl-ing: `http://<ip_address>:26657/status`.

   4. Set `pex` to `false`.
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
     switcheocli keys add minter --keyring-backend file
     ```

    Each wallet serves a different role:
    - `val`: Your validator consensus / application key.
    - `oraclewallet`: Oracle application key - validators are oracles at genesis and will need to submit oracle txns result. In future oracles will be incentivized separately.
    - `liquidator`: Liquidator application key - validators are liquidators at genesis and will need to submit liquidation txns. In future liquidators will be incentivized separately.
    - `minter`: Minter admin key - needed for testnet only.

4. Send SWTH to all wallets for self-staking and paying network fees. You can deposit NEP-5 SWTH into Switcheo TradeHub and then transfer SWTH from another wallet through the following command:

    `switcheocli tx bank send --from <from_name> --keyring-backend file -y -b block val <to_address> <amountInSatoshis>swth`

    Example to send 1000 SWTH from `mywallet` to `swth1haze3..n5e`:

    `switcheocli tx bank send --from mywallet --keyring-backend file -y -b block val swth1haze3ah2pwdhgstw9v5fcfphqccp359xgrgn5e 100000000000swth`

    If you are setting up a testnet node, contact one of a Switcheo support member for some fake SWTH!

## Running your node

Start your node using `switcheoctl`:

If you are starting a sentry or non-validator node, start it using:

`switcheoctl start -n`

If you are starting a validator node, start it using:

`switcheoctl start`

If you are running a validator node behind a sentry node, update your sentry node configuration to not gossip your validator node's IP:

```bash
# In the validator node(s):

# Get your node ID(s):
$ curl http://localhost:26657/status

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

# Comma separated list of peer IDs in `<node_id>@<ip_address>:26656` format to keep private (will not be gossiped to other peers), e.g:
private_peer_ids = "f09e200a655ce63e49b3710653258a674730e036@3.87.179.235:26656"

# Restart your sentry node:
$ switcheoctl restart -n
```

### Installing your node to a data directory

By default, the Switcheo TradeHub node is configured to run from the home directory of the installing user.

To run it from a different data directory, simply copy the node files into your data directory and symlink the copied files and folders.

For example to run from `/data`:

```bash
switcheoctl stop
cp ~/.switcheo* /data
ln -s /data/switcheocli  ~/.switcheocli
ln -s /data/switcheo_config ~/.switcheo_config
ln -s /data/switcheod ~/.switcheod
ln -s /data/switcheo_logs ~/.switcheo_logs
ln -s /data/switcheo_migrations ~/.switcheo_migrations
switcheoctl start
```

Ensure file and folder permissions remain unchanged.

## Stake as a validator

**:exclamation: WARNING :exclamation:**

**You should check that the node has caught up to latest block by running `switcheocli status` before continuing! If your node is still syncing (`sync_info.catching_up: true`), it will not be able to validate new blocks and you will end up getting slashed / jailed.**

Ensure your node is healthy to avoid getting your stake slashed. You can find info on your node at:

  `curl localhost:26657/abci_info`

You can check that your wallets have sufficient SWTH after starting through:

   `curl http://localhost:1318/auth/accounts/<address>`

You may need some time for your node to sync to see updated information.

Promote your node to validator with this command:

```bash
switcheoctl create-validator -a <amountToStakeInSatoshis>
```

You can check that is is successful by getting your validator address and looking up the chain:

```bash
switcheocli keys show val --keyring-backend file --bech val -a

curl http://localhost:1317/staking/validators/<val_address>
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
--chain-id=<chain_id> \
--keyring-backend file \
--yes
```

The validator details are similar to that of Cosmos Hub and can be found [here](https://hub.cosmos.network/master/validators/validator-setup.html#edit-validator-description).

---

## Validator Security

Each validator candidate is encouraged to run its operations independently, as diverse setups increase the resilience of the network. The following guide is adapted from the Cosmos Hub security page.

### Key Management

It is critical that an attacker cannot steal your validator key. If this is possible, it puts the entire stake delegated to the compromised validator at risk. Hardware security modules are a way to mitigate this risk.

HSM modules must support ed25519 signatures for the hub. Please see the [KMS guide](./KMS.md) for how you may use Tendermint KMS to interface with either the Ledger or YubiHMS2 hardware modules.

It is also important that only one instance of each validator node is active at any time. If two copies of a node is ran with the same validator key, the validator will likely sign the same block and be slashed. As such, any redundant setup where there is a backup node on standby must be configured with extreme care.

### Sentry Nodes (DDOS Protection)

Validators are responsible for ensuring that the network can sustain denial of service attacks.

One recommended way to mitigate these risks is for validators to carefully structure their network topology in a so-called [sentry node architecture](https://forum.cosmos.network/t/sentry-node-architecture-overview/454).

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

Validator nodes should only open the P2P port (26656) and rely on your sentry nodes for serving other requests. See the FAQ below for more information.

---

## FAQ

### Inbound traffic ports

#### Validator Node

The following ports should be open to allow for p2p traffic between nodes:

- 26656

#### Sentry Node

The following ports should be open to allow inbound query traffic:

- 1317 - Nginx reverse proxy to Cosmos SDK API and REST API
- 1318 - Cosmos SDK API (optional when using nginx proxy)
- 5000 - REST WS
- 5001 - REST API (optional when using nginx proxy)
- 26656 - P2P
- 26657 - Tendermint API (optional low level API)

### Chain ID

The chain IDs are already configured appropriately if you are using the correct package.

- Testnet: `switcheochain`
- Mainnet: `switcheo-tradehub-1`

### Unjail your jailed validator node

1. Ensure your node is not still syncing before unjailing or you will be slashed again!

2. Send a unjail txn:

    `switcheocli tx slashing unjail --from val --keyring-backend file -y -b block`

### Unstake your self-staked tokens

1. Get your validator address:

    `switcheocli keys show val --keyring-backend file --bech val -a`

2. Send an unstake txn:

    `switcheocli tx staking unbond <val_address> 100000000000swth --from val -y -b block --keyring-backend file`

### Stake additional tokens

1. Get your validator address:

    `switcheocli keys show val --keyring-backend file --bech val -a`

2. Send a stake txn:

    `switcheocli tx staking delegate <val_address> 100000000000swth --from val -y -b block --keyring-backend file`

## Debugging

Logs for all services / processes under Switcheo TradeHub are written to the .switcheo_logs folder.

You can tail all logs for debugging with:

`tail -f ~/.switcheo_logs/*`

To tail a specific service's log:

`tail -f ~/.switcheo_logs/start*`
