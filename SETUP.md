# Manual Setup

This guide contains steps to set up a node manually.

## Requirements

All nodes need the following databases installed regardless of type.

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
sudo bash -c 'cat <<EOT >> /etc/nginx/conf.d/switcheocli.conf
server {
    listen       5001 default_server;
    listen       [::]:5001 default_server;

    server_name  _;

    location / {
        proxy_pass  http://127.0.0.1:5002;
    }

    location /node_info {
        add_header 'Access-Control-Allow-Origin' '*';
        proxy_pass  http://127.0.0.1:1317;
    }

    location ~ ^/(staking|supply|slashing|distribution|auth|txs|subaccount|blocks)/ {
        add_header 'Access-Control-Allow-Origin' '*';
        proxy_pass  http://127.0.0.1:1317;
    }
}
EOT
'

# restart nginx or reload config
sudo systemctl restart nginx
```

Install jq, a lightweight command-line JSON processor:
```bash
sudo apt-get install -y jq
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
export TENDERMINT_SERVER_PORT=26657 # default
export COSMOS_REST_SERVER_HOST=0.0.0.0
export COSMOS_REST_SERVER_PORT=1317 # make sure this matches nginx
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
```

Source ~/.bashrc:
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
sed -i -e 's/laddr = "tcp:\/\/127.0.0.1:26657"/laddr = "tcp:\/\/0.0.0.0:26657"/g' ~/.switcheod/config/config.toml
sed -i -e 's/addr_book_strict = true/addr_book_strict = false/g' ~/.switcheod/config/config.toml

# if this is a validator node behind a sentry node, disable pex:
sed -i -e "s/pex.*/pex = false/g" ~/.switcheod/config/config.toml

# get peers and genesis file from a trusted node
# note that this configuration may differ if you are running in sentry configuration!

## testnet
persistent_peers=026b889e51c31c5370c4a6b43d7193d583efa4a2@54.255.42.175:26656
node_url=54.255.42.175:26657

## mainnet
persistent_peers=d363e17a3d4c7649e5c59bcd33176a476433108c@54.179.34.89:26656
node_url=54.179.34.89:26657

# set persistent_peers
cmd="s/persistent_peers = \"\"/persistent_peers = \"${persistent_peers}\"/g"
sed -i -e "${cmd}" ~/.switcheod/config/config.toml

# load genesis file
curl -s "${node_url}/genesis" | jq '.result.genesis' > ~/.switcheod/config/genesis.json

# initialise supporting directories
mkdir -p ~/.switcheo_logs/old ~/.switcheo_migrations/migrate ~/.switcheo_config

# initialise db
createdb switcheochain
```

### Running node

If running the `oracle` and `liquidator` services, ensure `WALLET_PASSWORD` is set.
The `start-all` command will attempt to run these subservices and load `oraclewallet` and `liquidator` accounts from the file-based keyring if it is present.

```bash
WALLET_PASSWORD=xxx switcheod start-all
# or
switcheod start-all
```

### Logs
You can read logs here:
```bash
tail -f ~/.switcheo_logs/*
```

### Promoting a node to a validator

```bash
# get node public key on validator machine
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
[program:switcheod]
user=${CURRENT_USER}
command=switcheod start-all
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
