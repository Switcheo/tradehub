# Manual Setup

This guide contains steps to set up a node manually.

## Architecture

Please read about the [sentry node architecture](./README.md#Sentry-Nodes-(DDOS-Protection)) for the recommended production environment. Generally two nodes, one validator and one sentry / public node should be used.

### Services

The `switcheod` binary contains a full Switcheo TradeHub node as well as several other services.

- `switcheod start` runs just the full node
- `switcheod rest-server` runs the cosmos-sdk REST server that allows for querying of chain data.
- `switcheod persistence` runs a service that indexes trade transactions into postgresql and redis.
- `switcheod interchain` runs a service that writes transaction hashes for deposits and withdrawals into the `persistence` database - run this if you need that data.
- `switcheod rest-api` runs a custom REST API server that serves queries from data indexed by the `persistence` service.
- `switcheod ws-api` runs a custom WS API server that streams data that is indexed by the `persistence` service.
- `switcheod oracle` runs the oracle service - validators need to run this when trading begins, however this can be done on a separate node (such as a sentry node). Additionally, `persistence` and `rest-api` need to be running.
- `switcheod liquidator` runs the liquidator service - only one validator needs to run this, but any validators that do will earn additional rewards. This can be done on a separate node (such as a sentry node).
- `switcheod relayer` runs a service that helps users to create Ethereum wallets and deposit transactions - this only needs to be run by exchange operators.
- `switcheod start-all -a` runs all the above services.

## Requirements

Nodes that serve public APIs (running `persistence` and `reset-api`, or `ws-api`) need the following databases installed.

Validators need at least one full node with these installed to run the oracle service, but can
skip installing these for the validator node itself.

- Redis
- Postgresql

## Setup Dependencies

Update package information:

```bash
sudo apt-get update
```

Install and configure redis like so:

```bash
# install redis
sudo apt-get install -y redis-server

# use systemd to manage redis
sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.$(date +%y%b%d-%H%M%S)
sudo sed -i -e 's/^supervised no/supervised systemd/g' /etc/redis/redis.conf
sudo systemctl restart redis.service
```

Install and configure postgresql like so:

```bash
# install psql
sudo apt-get install -y postgresql-10

# change authentication method to trust mode
# do not expose postgres on any public interfaces!
sudo sed -i -e '/^local   all             postgres                                peer$/d' \
    -e 's/ peer/ trust/g' \
    -e 's/ md5/ trust/g' \
    /etc/postgresql/10/main/pg_hba.conf

# restart for config reload
sudo service postgresql restart

# modify this if different
POSTGRES_USER=ubuntu

# make user a psql superuser
psql postgres -tAc "SELECT 1 FROM pg_roles WHERE rolname='${POSTGRES_USER}'" | grep -q 1 || sudo -i -u postgres psql -c "CREATE USER ${POSTGRES_USER} SUPERUSER;"
```

If you are running a public facing node, you should install nginx to serve requests:

```bash
# install nginx
sudo apt-get install -y nginx

# setup nginx proxy such that cosmos REST endpoints go to 1317
# while the rest goes to the custom off chain REST server at 5002.

# create nginx cache folder
sudo mkdir -p /var/cache/nginx

sudo bash -c 'cat <<EOT >> /etc/nginx/conf.d/switcheocli.conf
proxy_cache_path /var/cache/nginx/tradescan levels=1:2 keys_zone=tradescan_cache:10m max_size=10g inactive=2s use_temp_path=off;
proxy_cache_path /var/cache/nginx/cosmos_rest_server levels=1:2 keys_zone=cosmos_rest_server_cache:10m max_size=10g inactive=2s use_temp_path=off;
proxy_cache_path /var/cache/nginx/tendermint_api_server levels=1:2 keys_zone=tendermint_api_server_cache:10m max_size=10g inactive=2s use_temp_path=off;

server {
    listen       5001 default_server;
    listen       [::]:5001 default_server;

    server_name  _;

    proxy_cache tradescan_cache;
    proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
    proxy_cache_valid any 1s;
    proxy_cache_lock on;

    location / {
        proxy_pass  http://127.0.0.1:5002;
    }

    location /node_info {
        add_header 'Access-Control-Allow-Origin' '*';
        proxy_pass  http://127.0.0.1:1317;
    }

    location ~ ^/(staking|supply|slashing|distribution|auth|txs|subaccount|blocks|bank)/ {
        add_header 'Access-Control-Allow-Origin' '*';
        proxy_pass  http://127.0.0.1:1317;
    }
}

server {
    listen       1317 default_server;
    listen       [::]:1317 default_server;

    server_name  _;

    proxy_cache cosmos_rest_server_cache;
    proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
    proxy_cache_valid any 1s;
    proxy_cache_lock on;

    location / {
        proxy_pass  http://127.0.0.1:1318;
    }
}

server {
    listen       26657 default_server;
    listen       [::]:26657 default_server;

    server_name  _;

    proxy_cache tendermint_api_server_cache;
    proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
    proxy_cache_valid any 1s;
    proxy_cache_lock on;

    location / {
        proxy_pass  http://127.0.0.1:26659;
    }
}
EOT
'

# restart nginx or reload config
sudo systemctl restart nginx
```

## Setup Node

### Download latest release

Release binaries can be found [here](./README.md#download-a-switcheoctl-release). Unzip them to your home directory instructed in the main guide.

Copy switcheod and switcheocli to /usr/local/bin

```bash
cd install-mainnet && sudo cp bin/switcheod bin/switcheocli /usr/local/bin && cd - && rm -rf install-mainnet
```

### Configure environment

Add these to `.bashrc` or otherwise ensure these env variables are applied to the context of the node.

```bash
export APP_ENV=production
export CHAIN_ID=switcheo-tradehub-1 # or "switcheochain" for testnet
export TENDERMINT_SERVER_HOST=localhost
export TENDERMINT_SERVER_PORT=26659 # default
export COSMOS_REST_SERVER_HOST=0.0.0.0
export COSMOS_REST_SERVER_PORT=1318 # make sure this matches nginx
export API_REST_PORT=5002 # make sure this matches nginx
export POSTGRES_HOST=localhost
export POSTGRES_PORT=5432
export POSTGRES_DB=switcheochain
export POSTGRES_USER=ubuntu # modify if different
export REDIS_HOST=localhost
export REDIS_PORT=6379
export LOG_TO_FILE=1
export LOG_LEVEL=info
export SWTH_UPGRADER=cosmos1flqfs2dzzkrf49aj5pg0nj340jdqzgje02smww
export SEND_ETH_TXNS=1
export MINFDS=1024
```

If using `.bashrc`, remember to reload it with:

```bash
source ~/.bashrc
```

### Configure node

```bash
# initialize validators's and node's configuration files. Change <moniker> to your identifier.
switcheod init <moniker> --chain-id $CHAIN_ID

# configure switcheocli
switcheocli config chain-id $CHAIN_ID
switcheocli config output json
switcheocli config indent true
switcheocli config trust-node true

# configure node timeout to 1s,:
sed -i -e 's/timeout_commit = "5s"/timeout_commit = "'"1s"'"/g' ~/.switcheod/config/config.toml

# configure node pruning settings to keep-recent 100 keep-every 10000 interval 10:
sed -i -e 's/pruning-keep-recent = "0"/pruning-keep-recent = "100"/g' ~/.switcheod/config/app.toml
sed -i -e 's/pruning-keep-every = "0"/pruning-keep-every = "10000"/g' ~/.switcheod/config/app.toml
sed -i -e 's/pruning-interval = "0"/pruning-interval = "10"/g' ~/.switcheod/config/app.toml

# if this is a public node, listen on public interfaces and disable CORS:
sed -i -e 's/cors_allowed_origins = \[\]/cors_allowed_origins = \["*"\]/g' ~/.switcheod/config/config.toml
sed -i -e 's/laddr = "tcp:\/\/127.0.0.1:26657"/laddr = "tcp:\/\/0.0.0.0:26659"/g' ~/.switcheod/config/config.toml
sed -i -e 's/addr_book_strict = true/addr_book_strict = false/g' ~/.switcheod/config/config.toml

# if this is a validator node behind a sentry node, disable pex:
sed -i -e "s/pex.*/pex = false/g" ~/.switcheod/config/config.toml

# get peers and genesis file from a trusted node
# note that this configuration may differ if you are running in sentry configuration!

## testnet
persistent_peers=026b889e51c31c5370c4a6b43d7193d583efa4a2@54.255.42.175:26656
node_url=54.255.42.175:26657

## mainnet
persistent_peers=d363e17a3d4c7649e5c59bcd33176a476433108c@54.255.5.46:26656
node_url=54.179.34.89:26657

# set persistent_peers
cmd="s/persistent_peers = \"\"/persistent_peers = \"${persistent_peers}\"/g"
sed -i -e "${cmd}" ~/.switcheod/config/config.toml

# load genesis file
apt-get install jq -y # next cmd uses jq
curl -s "${node_url}/genesis" | jq '.result.genesis' > ~/.switcheod/config/genesis.json

# initialise supporting directories
mkdir -p ~/.switcheo_logs/old ~/.switcheo_migrations/migrate ~/.switcheo_config

# initialise db
createdb switcheochain
```

### Running node

If running the `oracle` and `liquidator` services, ensure `WALLET_PASSWORD` is set.
The `start-all -a` command will attempt to run these subservices and load `oraclewallet` and `liquidator` accounts from the file-based keyring if it is present.

```bash
# run node only (for validator that is running oracle / liquidator service separately)
switcheod start
# run node with public apis
switcheod start-all -a
# run node with public apis and oracle and liquidator services
WALLET_PASSWORD=xxx switcheod start-all -a
```

### Logging and debugging

You can find the logs from `switcheod` here:

```bash
tail -f ~/.switcheo_logs/*
```

### Stake as a validator

**:exclamation: WARNING :exclamation:**

**You should check that the node has caught up to latest block by running `switcheocli status` before continuing! If your node is still syncing (`sync_info.catching_up: true`), it will not be able to validate new blocks and you will end up getting slashed / jailed.**

Ensure your node is healthy to avoid getting your stake slashed. You can find info on your node at:

  `curl localhost:26657/abci_info`

You can check that your wallets have sufficient SWTH after starting through:

   `curl http://localhost:5001/auth/accounts/<address>`

You may need some time for your node to sync to see updated information.

Promote your node to validator with this command:

```bash
# get node public key on validator machine
# testnet: switcheod node tendermint show-validator
$ switcheod tendermint show-validator
swthvalconspub1zcjduepqqsuvl3xj58qmfv49je....

# on operator machine
switcheocli tx staking create-validator --amount <amountToStakeInSatoshis>swth --moniker <yourmoniker> --pubkey <pubkey> --commission-max-change-rate 0.010000000000000000 --commission-max-rate 0.200000000000000000 --commission-rate 0.100000000000000000 --min-self-delegation 1 --fees 100000000swth -b block --from val --keyring-backend file -y
```

## Setup auxilliary services

You should strongly consider setting up these services to maintain the well-being of your node.

### Supervisor

You can use supervisord or systemd to ensure your node remains up. Here is an example supervisor configuration.

```config
[supervisord]
environment=WALLET_PASSWORD="the-keyring-password" # only if running all services
minfds=1024
[program:switcheod]
user=${CURRENT_USER}
command=switcheod start-all -a
autostart=true
autorestart=true
startretries=100
startsecs=10
stopasgroup=true
killasgroup=true
stopsignal=INT
stderr_logfile=/var/log/supervisor/switcheod.err.log
stdout_logfile=/var/log/supervisor/switcheod.out.log
```

> Important: When running the `oracle` and `liquidator` service, the `oraclewallet` and `liquidator` wallet must be in the file-based keyring of the same machine, and the `WALLET_PASSWORD` environment must be set. Otherwise, `WALLET_PASSWORD` should not be set.

### Logrotate

Switcheo TradeHub outputs quite alot of logs, and it may be neccessary to keep them. Use logrotate to manage the disk usage of logs.

```bash
sudo bash -c 'cat <<EOT >> /etc/logrotate.d/switcheo_logs
$HOME/.switcheo_logs/*.log ${HOME}/.switcheo_logs/*.err {
    olddir ${HOME}/.switcheo_logs/old
    daily
    missingok
    rotate 30
    compress
    copytruncate
    notifempty
}
EOT
```
