---
name: supra-fund-and-deploy-on-testnet
description: Stand up a working Supra testnet account and publish a Move package — generate a profile, fund it from the documented faucet endpoint, compile, simulate, publish, then verify the module landed.
api: Supra RPC Node API
base_url: https://rpc-testnet.supra.com
operations:
  - chain_id
  - faucet_v1
  - faucet_v2
  - get_account_v3
  - transaction_parameter
  - simulate_txn_v3
  - get_account_modules_v3
  - get_account_module_v3
  - view_function_v3
generated: '2026-08-29'
method: generated
source: >-
  openapi/supra-rpc-node-openapi.yml for the operationIds, https://docs.supra.com/network-information for the
  testnet RPC URL and faucet path, and https://docs.supra.com/network/move/cli-commands for the CLI commands.
---

# Fund a Supra testnet account and deploy a Move package

Supra's test environment is a real public network, not a mock: same software, same contract,
different chain id. Switching between test and production is one base URL — there are no
test keys to swap, because the API takes no keys at all.

| | Mainnet | Testnet |
|---|---|---|
| RPC URL | `https://rpc-mainnet.supra.com` | `https://rpc-testnet.supra.com` |
| Chain id | `8` | `6` |
| Faucet | none | yes |

## 1. Confirm you are on testnet before anything else

    GET /rpc/v1/transactions/chain_id    # chain_id

Must return `6`. If it returns `8` you are pointed at mainnet — stop.

## 2. Create a key profile (CLI)

The Supra CLI ships as a Docker image; see `cli/supra-cli.yml`.

    supra key generate-profile myAccount
    supra key activate-profile myAccount
    supra key list-profile

Pass `--profile myAccount` on every later command. Omitting it is the documented failure
mode — Supra's own published Agent Skill changelog records that subsequent commands fail
with "profile not found" without it.

## 3. Fund it

Either through the CLI:

    supra move account fund-with-faucet --profile myAccount --url https://rpc-testnet.supra.com

or directly against the documented faucet endpoint:

    GET /rpc/v1/wallet/faucet/{address}    # faucet_v1

The faucet is an ordinary GET on the RPC host and is part of the OpenAPI contract, not a
separate console. Look up the resulting funding transaction with:

    GET /rpc/v2/wallet/faucet/transactions/{hash}   # faucet_v2

Do not use `faucet_transaction_by_hash_v1` — it is marked `deprecated: true`.

## 4. Verify the account exists and is funded

    GET /rpc/v3/accounts/{address}       # get_account_v3

Returns `sequence_number` and `authentication_key`. A brand-new account starts at
`sequence_number: 0`. Balance is a Move resource, so read it with:

    GET /rpc/v3/accounts/{address}/resources/{resource_type}   # get_account_resource_v3

using the URL-encoded `CoinStore` struct tag, or via the CLI:

    supra move account balance --profile myAccount --url https://rpc-testnet.supra.com

## 5. Compile, simulate, publish

    supra move tool compile --package-dir <PACKAGE_DIR>
    supra move tool test --package-dir <PACKAGE_DIR>
    supra move tool publish --profile myAccount --package-dir <PACKAGE_DIR> --url https://rpc-testnet.supra.com

Publishing is a transaction. Rehearse it first with `POST /rpc/v3/transactions/simulate`
(`simulate_txn_v3`) and check `transaction_parameter` for the current gas and expiry bounds.
Even on testnet, get in the habit: on mainnet this call is irreversible.

## 6. Verify the module actually landed

    GET /rpc/v3/accounts/{address}/modules                 # get_account_modules_v3
    GET /rpc/v3/accounts/{address}/modules/{module_name}   # get_account_module_v3

`get_account_module_v3` returns `MoveModuleBytecode` with its `abi` — read that ABI to
confirm the functions you expect are exposed and which are `view`.

Then exercise a read path without spending gas:

    POST /rpc/v3/view                     # view_function_v3

Body is a `ViewRequest`: a fully qualified `address::module::function`, plus
`type_arguments` and `arguments`. Returns `MoveValue`s.

## Note on SupraEVM

`https://docs.supra.com/network-information` lists SupraEVM's REST API and chain id as
**TBA** and its faucet as "Coming soon" as of 2026-08-29. This skill is MoveVM only.
