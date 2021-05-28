<!--
Guiding Principles:

Changelogs are for humans, not machines.
There should be an entry for every single version.
The same types of changes should be grouped.
Versions and sections should be linkable.
The latest version comes first.
The release date of each version is displayed.
Mention whether you follow Semantic Versioning.

Usage:

Change log entries are to be added to the Unreleased section under the
appropriate stanza (see below). Each entry should ideally include a tag and
the Github issue reference in the following format:

* (<tag>) \#<issue-number> message

The issue numbers will later be link-ified during the release process so you do
not have to worry about including a link manually, but you can if you wish.

Types of changes (Stanzas):

"Features" for new features.
"Improvements" for changes in existing functionality.
"Deprecated" for soon-to-be removed features.
"Bug Fixes" for any bug fixes.
"Client Breaking" for breaking CLI commands and REST routes used by end-users.
"API Breaking" for breaking exported APIs used by developers building on SDK.
"State Machine Breaking" for any changes that result in a different AppState given same genesisState and txList.
Ref: https://keepachangelog.com/en/1.0.0/
-->


## [v1.11.1](https://github.com/Switcheo/tradehub/releases/tag/v1.11.1) - 2021-01-18
### Improvements
- Add subaccount query, see oracle readme.
- Build binary with cleveldb adapter. Existing nodes that have started with goleveldb (default) will not be able to use cleveldb, but must still install leveldb package in their machine as the binary will require those headers. Only fresh installation will use cleveldb. You can check your `db_backend` in `~/.switcheod/config/config.toml`.

### Bug Fixes
- Fix API to last price in `/get_market_stats` endpoint, and ws `get_market_stats`, `market_stats` channels, where decimals places was shifted wrongly.
- Fix recent trades in ws `get_recent_trades` and `recent_trades` channels.

## [v1.12.0](https://github.com/Switcheo/tradehub/releases/tag/v1.12.0) - 2021-01-29

### Improvements
- Purge orders from state that have reached end state - "filled", "cancelled". v1.12.0 migration will remove all orders from orders store that have reached end state.
-  Save oracle votes to db for better analytics.
- AMM will now quote up to half the reserves.
### Bug Fixes
- Fix AMM not requoting orderbooks when swap fee changes
- Fix cancelled stop orders still existing in account's open orders. v1.12.0 migration will remove all orders that have reached end state from accounts' open orders.
- Fix allocated margin being added to untriggered stop-limit orders during change leverage.

## [v1.14.0](https://github.com/Switcheo/tradehub/releases/tag/v1.14.0) - 2021-02-16

### Improvements
- Limit the pruning of oracles votes to 1000 per block per oracle, in order to avoid a situation that can potentially cause slow blocks to occur.
### Bug Fixes
- Fix rounding direction in AMM quoting implementation.


## [v1.15.0](https://github.com/Switcheo/tradehub/releases/tag/v1.15.0) - 2021-03-15

### Improvements
- Add native withdrawal of SWTH to bsc
- Add bsc fee payer and external event tracking services
- Add state queries for locked user coins
- Use [gasnow.org](http://gasnow.org/) for estimating gas prices in fee service
### Bug Fixes
- Fix `get_market_stats` REST endpoint giving wrong result
- Fix possible panic in websocket service for some invalid parameters
- Fix panic in `SetCommitmentCurve` message.

## [v1.16.0](https://github.com/Switcheo/tradehub/releases/tag/v1.16.0) - 2021-03-23

### Bug Fixes

- Fix edge case scenarios causing stucked order margin
- Fix possible panic in linking pool when it is already linked to another market

## [v1.17.0](https://github.com/Switcheo/tradehub/releases/tag/v1.17.0) - 2021-05-24

### Bug Fixes

- Fix export/init genesis state
