# KMS

This guide describes how to run your Switcheo TradeHub validator node with a Key Management System (KMS) using [Tendermint KMS (tmkms)](https://github.com/iqlusioninc/tmkms).

Please read the tmkms documentation thoroughly to have an understanding on how it works.

## Architecture

The KMS may be installed either on the same machine as your validator node, or in a separate signing machine that is connected via a private network.

Running the KMS service on a separate machine allows the validator keys to survive compromise of the validator host, and also gives some protection to double-signing, in the event multiple validator proesses are ran.

## Dependencies

- Linux
- [Rust (stable) / Cargo](https://rustup.rs)
- Ledger or YubiHMS2 hardware (may use `softsign` for testing purposes)
- `libusb` for USB support
- `build-essential` for compiling tmkms
- `tmkms`

## Installation

Install dependencies:

```bash
# install libusb
sudo apt install libusb-1.0-0-dev -y
# install build packages for compiling tmkms
sudo apt install build-essential libudev-dev pkg-config -y
# install rust / cargo (see: https://rustup.rs/)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Install `tmkms` using Cargo. Set `--features=yubihsm` or `--features=ledgertm` to only install support for those signature providers. See also [yubihsm-server](https://github.com/iqlusioninc/tmkms/blob/develop/README.yubihsm.md#yubihsm-server-feature) for additional administration options.

```bash
cargo install tmkms cargo install tmkms --features=yubihsm,ledgertm,softsign --version=0.8.0
```

This may take awhile.

### YubiHSM

If using YubiHSM, you will need to grant `tmkms` access. See [udev configuration](https://github.com/iqlusioninc/tmkms/blob/develop/README.yubihsm.md#udev-configuration) for more info.

Follow the [production YubiHSM guide](https://github.com/iqlusioninc/tmkms/blob/develop/README.yubihsm.md#production-yubihsm-2-setup) in the `tmkms` repo for further setup instructions.

## Setup

1. Initialize `tmkms` to a secure directory:

    ```bash
    tmkms init /path-to-kms-directory
    ```

2. Copy your validator key to your KMS node / path.

    Here's an example using a single machine and `softsign`:

    ```bash
    tmkms softsign import ~/.switcheod/config/priv_validator_key.json ~/kms/secrets/switcheochain-consensus.key
    ```

     Otherwise copy over the `priv_validator_key.json` file first, and use the appropriate signer subcommand.


3. Configure KMS for Switcheo TradeHub by modifying `/path-to-kms-dir/tmkms.toml` as such:

    ```toml
    # Tendermint KMS configuration file

    ## Chain Configuration

    ### Switcheo TradeHub Network

    [[chain]]
    id = "switcheochain"
    key_format = { type = "bech32", account_key_prefix = "swthpub", consensus_key_prefix = "swthvalconspub" }
    state_file = "/path-to-kms-dir/state/switcheo-tradehub-1-consensus.json"

    ## Signing Provider Configuration

    ### YubiHSM2 Provider Configuration

    # comment out this section if not using YubiHSM2
    [[providers.yubihsm]]
    adapter = { type = "usb" }
    # password can be used directly using password=xxx instead of password_file
    auth = { key = 1, password_file = "/path-to-kms-dir/secrets/yubihsm-password.txt" }
    # serial_number = "0123456789" # serial number of a specific YubiHSM to connect to (optional)
    keys = [
        { key = 1, type = "consensus", chain_ids = ["switcheochain"] },
    ]

    ### Ledger Provider Configuration

    # comment out this section if not using Ledger
    [[providers.ledgertm]]
    chain_ids = ["switcheochain"]

    ### Software-based Signer Configuration

    # comment out this section if not using softsign
    [[providers.softsign]]
    chain_ids = ["switcheochain"]
    key_type = "consensus"
    path = "/home/ubuntu/kms/secrets/switcheochain-consensus.key"

    ## Validator Configuration

    [[validator]]
    chain_id = "switcheochain"
    addr = "tcp://<optional-peer-id>@<validator-ip or localhost>:26658"
    secret_key = "/path-to-kms-dir/secrets/kms-identity.key"
    protocol_version = "v0.33"
    reconnect = true
    ```

    For more options, check this example config:
    [https://github.com/iqlusioninc/tmkms/blob/develop/tmkms.toml.example](https://github.com/iqlusioninc/tmkms/blob/develop/tmkms.toml.example)

4. Run tmkms

    ```bash
    tmkms start -c /path-to-kms-dir/tmkms.toml
    ```

    At this point, it should try to connect to the tendermint port of the Switcheo TradeHub node but fail - that is to be expected.

    ```bash
    ERROR tmkms::client: [switcheochain@tcp://localhost:26658] protocol error
    ```

5. Update your validator config to use the KMS service instead of local private key.

    ```bash
    vi ~/.switcheod/config/config.toml
    ```

    ```toml
    priv_validator_laddr = "tcp://0.0.0.0:26658"
    ```

6. Restart your validator node.

    ```bash
    switcheoctl restart
    ```

7. You should now see signing information on `tmkms`.

    ```bash
    Aug 06 10:26:44.398  INFO tmkms::session: [switcheochain@tcp://localhost:26658] signed PreVote:36D89B049E at h/r/s 10209/0/1 (0 ms)
    Aug 06 10:26:44.403  INFO tmkms::session: [switcheochain@tcp://localhost:26658] signed PreCommit:36D89B049E at h/r/s 10209/0/2 (0 ms)
    ```

8. You may want to add `tmkms` to supervisor:

    ```bash
    sudo vi /etc/supervisor/conf.d/tmkms.conf
    ```

    ```conf
    [program:tmkms]
    user=ubuntu
    command=/home/ubuntu/.cargo/bin/tmkms start -c /path-to-kms-dir/tmkms.toml
    autostart=true
    autorestart=true
    startretries=100
    startsecs=10
    stopasgroup=true
    killasgroup=true
    stopsignal=INT
    stderr_logfile=/var/log/supervisor/tmkms.err.log
    stdout_logfile=/var/log/supervisor/tmkms.out.log
    ```

9. You may now remove the validator key file from your validator machine:

    ```bash
    rm ~/.switcheod/config/priv_validator_key.json
    ```

## References

- [https://github.com/iqlusioninc/tmkms](tmkms)
- [KMS_YubiHSM Set up l Cosmos Hub](https://medium.com/node-a-team/kms-yubihsm-set-up-l-cosmos-hub-c4a83ffbecd3)
