# Getting started

## Requirements

You will need a Linux, Ubuntu 18 compute instance for running your Switcheo TradeHub node.

Switcheo TradeHub installation only works on Ubuntu 18 at the moment, other Linux flavors will be available in the near future.

Here are the minimum system requirements for a validator node:

**Testnet**: 8GB RAM, 2 CPUs, 200GB Storage

## Download a `switcheoctl` release

The following package contains the Switcheo TradeHub client, as well as various tools
that may be useful in running a validator or seed node: [https://github.com/Switcheo/tradehub/releases](https://github.com/Switcheo/tradehub/releases)

You may use the following command to download and unzip the release:

```bash
curl -L https://github.com/Switcheo/tradehub/releases/download/v2.0.0-beta/install-testnet.tar.gz | tar -xz
```

## Install `switcheoctl`

`switcheoctl` allows you to install and control your Switcheo TradeHub node easily.

Install it with the following command:

```bash
cd install-testnet && sudo ./install.sh && cd - && rm -rf install-testnet
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
3. Restart shell: `exec $SHELL`

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
3. Restart shell: `exec $SHELL`
4. Create the required wallets for running a validator node. **Store the generated mnemonics in a safe, offline location!**

     ```bash
     # You will need to choose a secure password when adding the first wallet.
     # All other wallets in the keyring must use the same password.
     # Store the generated mnemonics for each wallet as a backup!
     switcheocli keys add val --keyring-backend file
     switcheocli keys add oraclewallet --keyring-backend file
     switcheocli keys add liquidator --keyring-backend file
     ```

    Each wallet serves a different role:
    - `val`: Your validator operator wallet.
    - `oraclewallet`: Oracle subaccount wallet - validators are oracles at genesis and will need to submit oracle result txns when trading begins. In future oracles will be incentivized through a separate incentive model.
    - `liquidator`: Liquidator subaccount wallet - validators are liquidators at genesis and will need to submit liquidation txns when trading begins. In future liquidators will be incentivized through a separate incentive model.

    The oracle and liquidator services require public HTTP access to run. If your validator machine does not have such access, you should create those wallets on another machine running a public node (same as sentry node configuraton). These wallets must be bound as subaccounts to the main validator operator wallet through the `subaccounts` command / transaction, more info [here](#link-your-wallets-through-subaccounts). **If you do so, you should also create the `val` wallet on a separate machine, and ensure that you do not set any wallet password configuration on supervisord or switcheoctl (leave empty went prompted).**

    The oracle and liquidator services can be ran separately with the `switcheod oracle` and `switcheod liquidator` commands. However, these will do nothing until trading begins.

5. Send SWTH to all wallets for self-staking and paying network fees. You can deposit NEP-5 SWTH into Switcheo TradeHub and then transfer SWTH from another wallet through the following command:

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
