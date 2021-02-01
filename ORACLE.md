# Oracle

The oracle services are needed to support futures trading. Ensure that the oracle service is running, and wallets are linked properly through subaccounts, by following the guide below. The oracle wallet name should be named as `oraclewallet`.

<br>

## Check that you have an oracle wallet

You may choose to run the oracle service in either the validator or sentry node. If you followed the validator node setup guide, the oracle wallet is already created with the name `oraclewallet`. The oracle account will only be created if someone send funds to it.

If the oracle service is run in the sentry node, also create a dummy `val` wallet. This will not be needed in a future patch.

1. Check that val and oraclewallet exist

    ```bash
    switcheocli keys show oraclewallet --keyring-backend file
    switcheocli keys show val --keyring-backend file
    ```

2. Otherwise create an oraclewallet and send some funds to it to pay network fees. Create a dummy val wallet but don't send funds to it.

    ```bash
    switcheocli keys add oraclewallet --keyring-backend file
    ```

    ```bash
    # send 10 swth from val to oraclewallet
    switcheocli tx bank send --from val --keyring-backend file -y --fees 100000000swth -b block val <oraclewallet-address> 1000000000swth
    ```

    ```bash
    # there should be non-zero account_number
    switcheocli query account <oraclewallet-address>
    ```

    ```bash
    # no need to send funds to it. switcheoctl checks if val wallet is readable, otherwise there's an incorrect password error
    switcheocli keys add val --keyring-backend file
    ```

## Running the oracle service

1. Check that oracle service is running

    ```bash
    ps aux | grep "switcheod oracle"
    ```

2. If it's not running, check that there is a systemd service named `switcheod-oracle.service` and restart switcheo services. If your oraclewallet is in sentry node, you need need to restart it without `-n` flag. The oracle service requires access to oraclewallet so you need to provide the wallet password.

    ```bash
    ls /etc/systemd/system | grep switcheod

    switcheoctl restart
    ```

<br>

## Link your wallets through subaccounts

The `oraclewallet` should be linked to your `val` wallet as a subaccount, this will allow the `oraclewallet` to send oracle votes on behalf of your `val` wallet.
Before linking the wallets, it should be ensured that both the `val` and `oraclewallet` addresses have sufficient funds to pay fees.

1. Create the `oraclewallet` as a subaccount of your `val` wallet:

    Run this command in the node where your `val` wallet is.

    ```bash
    switcheocli tx subaccount create-sub-account --from val --keyring-backend file -y --fees 100000000swth -b block val <oraclewallet-swth-address> <val-swth-address>
    ```

2. Activate the subaccount:

    Run this command in the node where your `oraclewallet` wallet is.

    ```bash
    switcheocli tx subaccount activate-sub-account --from oraclewallet --keyring-backend file -y --fees 100000000swth -b block oraclewallet <oraclewallet-swth-address> <val-swth-address>
    ```

<br>

## Check that oracle wallet subaccount is linked properly

1. List your wallets and find your oracle wallet address

    ```bash
    switcheocli keys list --keyring-backend file

    #  {
    #    "name": "oraclewallet",
    #    "type": "local",
    #    "address": "swth1x7nw6vkkwgnzlchkw9q0ma5hn3nqkcx4d3d5nc",
    #  },
    ```

2. Check if oracle wallet is linked properly

    ```bash
    switcheocli query subaccount get <oraclewallet-address>

    # switcheocli query subaccount get swth1x7nw6vkkwgnzlchkw9q0ma5hn3nqkcx4d3d5nc
    #
    # {
    #   "main_account": "swth1kcez9p5t5cp80estaw8j97484g44d83w5d3ru9",
    #   "active": true
    # }
    ```

3. Check oracle service logs

    ```bash
    tail -f ~/.switcheod/logs/oracle*

    # most recent lines is healthy
    # ...
    # connected subaccount:  oraclewallet
    # wallet connected, running oracle node
    ```
