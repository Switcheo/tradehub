# Getting started

## Requirements

You will need a Linux compute instance for running your Switcheo TradeHub node.

Switcheo TradeHub has only been tested on Ubuntu 18 at the moment, but it should work for all Linux flavors. Both a VPS or bare-metal machine will work, but [security considerations](#Validator-Security) should be taken into account.

### Digital Ocean example

1. Create an account
2. Spin up an instance https://www.digitalocean.com/docs/droplets/quickstart/
    1. Go to Manage - Droplets - Create Droplet
    2. Choose a distribution - Ubuntu 18.04.3 (LTS) x64
    3. Choose a plan - Standard $10/mo
    4. Choose a datacenter region - Singapore (1)
    5. Authentication - [New SSH Key]((https://help.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)) (or use an existing one)
    6. Choose a hostname - e.g. `tradehub-validator`
3. Once your instance has spun up, ssh to it using it's IP address:

    `ssh root@<ip_address>`

## Download a `switcheoctl` release

The following package contains the Switcheo TradeHub client, as well as various tools
that may be useful in running a validator or seed node: https://github.com/ConjurTech/tradehub/releases

You can use the following convenience install script:

```bash
curl -L https://github.com/ConjurTech/tradehub/releases/download/v0.0.1/install.tar.gz | tar -xz
```

## Install `switcheoctl`

`switcheoctl` allows you to install and control your Switcheo TradeHub node easily.

Install it with the following command:

```bash
cd install && sudo ./install.sh && cd - && rm -rf install
```

## Set up a validator node

1. Update the validator config file that can be found at `/etc/switcheoctl/config/config.json`:
   1. `sudo nano /etc/switcheoctl/config/config.json`
   2. Choose a unique monikier that identifies you well and replace `hikaru` with it.
2. Install with: `switcheoctl install-validator`
3. Create the required wallets for running a validator node. **Store the generated mnemonics in a safe, offline location!**

     ```bash
     switcheocli keys add val --keyring-backend file
     # You will need to choose a secure password when adding the first wallet.
     # All other wallets in the keyring must use the same password.
     # Store the generated mnemonics for each wallet as a backup!
     switcheocli keys add oraclewallet --keyring-backend file
     switcheocli keys add interchainwallet --keyring-backend file
     switcheocli keys add minter --keyring-backend file
     switcheocli keys add liquidator --keyring-backend file
     ```

4. Send SWTH to all wallets for self-staking and paying network fees. You can deposit NEP-5 SWTH into Switcheo TradeHub and then transfer SWTH from another wallet through the following command:

    `switcheocli tx bank send --from <from_name> --keyring-backend file -y -b block val <to_address> <amountInSatoshis>swth`

    Example to send 1000 SWTH from `mywallet` to `swth1haze3..n5e`:

    `switcheocli tx bank send --from mywallet --keyring-backend file -y -b block val swth1haze3ah2pwdhgstw9v5fcfphqccp359xgrgn5e 100000000000swth`

## Run validator node

You're all set to run the validator node!

1. Start your node using `switcheoctl`:

    `switcheoctl start`

2. Ensure your node is healthy to avoid getting your stake slashed. You can find info on your node at:

    `curl localhost:26657/abci_info`


3. You can check that your wallets have sufficient SWTH after starting through:

   `curl http://localhost:1318/auth/accounts/<address>`

   You may need some time for your node to sync to see updated information.

## Stake as a validator

You should check that the node has caught up to latest block by running `switcheocli status`.
If your node is lagging behind, you may end up getting slashed and jailed.

Promote your node to validator with this command:

```bash
switcheoctl create-validator -a <amountToStakeInSatoshis>
```

You can check that is is successful by getting your validator address and looking up the chain:

```bash
switcheocli keys show val --keyring-backend file --bech val -a

curl http://localhost:1317/staking/validators/<val_address>
```

---

## Validator Security

Each validator candidate is encouraged to run its operations independently, as diverse setups increase the resilience of the network. The following guide is adapted from the Cosmos Hub security page.

### Key Management

It is critical that an attacker cannot steal your validator key. If this is possible, it puts the entire stake delegated to the compromised validator at risk. Hardware security modules are a way to mitigate this risk.

HSM modules must support ed25519 signatures for the hub. The YubiHSM can protect a private key but cannot ensure in a secure setting that it won't sign the same block twice.

The Tendermint team is also working on extending the Ledger Nano S application to support validator signing. This app can store recent blocks and mitigate double signing attacks.

It is also important that only one instance of each validator node is active at any time. If two copies of a node is ran with the same validator key, the validator will likely sign the same block and be slashed. As such, any redundant setup where there is a backup node on standby must be configured with extreme care.

### Sentry Nodes (DDOS Protection)

Validators are responsible for ensuring that the network can sustain denial of service attacks.

One recommended way to mitigate these risks is for validators to carefully structure their network topology in a so-called sentry node architecture.

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

### Recommended specifications

Mainnet: 8GB memory, 4vCPUs, 5TB storage.

### Inbound traffic ports

#### Sentry Node

#### Validator Node

The following ports should be open to allow for p2p traffic between nodes:

- 26656

The following ports should be open to allow inbound query traffic:

- 1317 - Nginx reverse proxy to Cosmos SDK API and REST API
- 1318 - Cosmos SDK API (optional when using nginx proxy)
- 5000 - REST WS
- 5001 - REST API (optional when using nginx proxy)
- 26656 - P2P
- 26657 - Tendermint API (optional low level API)

### Unjail your jailed validator node

1. Ensure your node is not still syncing before unjailing or you will be slashed again!

2. Send a unjail txn:

    `switcheocli tx slashing unjail --from val --keyring-backend file -y -b block`

### Unstake your self-staked tokens

1. Get your validator address:

    `switcheocli keys show val --keyring-backend file --bech val -a`

2. Send an unstake txn:

    `switcheocli tx staking unbond <val_address> 100000000000swth --from val -y -b block --keyring-backend file`

### Stake more tokens


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

