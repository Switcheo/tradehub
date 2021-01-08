# Oracle

The oracle services are needed to support futures trading. Ensure that the oracle service is running, and wallets are linked properly through subaccounts, by following the guide below.

<br>

## Running the oracle service

You may choose to run the oracle service in either the validator or sentry node. If you followed the validator node setup guide, the oracle wallet is created with the name `oraclewallet`. Ensure that the oracle account is created by sending an initial amount to it.

1. Check that oracle service is running
  ```bash
  ps aux | grep "switcheod oracle"
  ```

<br>

## Link your wallets through subaccounts

The `oraclewallet` should be linked to your `val` wallet as a subaccount, this will allow the `oraclewallet` to send oracle votes on behalf of your `val` wallet.
Before linking the wallets, it should be ensured that both the `val` and `oraclewallet` addresses have sufficient funds to pay fees.

Run these commands in your validator node, where your `val` wallet is.

1. To link the `oraclewallet` as a subaccount of your `val` wallet, you can use the following cli commands:

  ```bash
  switcheocli tx subaccount create-sub-account --from val --keyring-backend file -y --fees 100000000swth -b block val <oraclewallet-swth-address> <val-swth-address>
  switcheocli tx subaccount activate-sub-account --from oraclewallet --keyring-backend file -y --fees 100000000swth -b block oraclewallet <oraclewallet-swth-address> <val-swth-address>
  ```

2. You should run `switcheoctl restart` after linking your wallets, this will allow the oracle service to start correctly.

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
  switcheocli query subaccount get <oracle-wallet-address>

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

  # ...
  # connected subaccount:  oraclewallet
  # wallet connected, running oracle node
  ```
