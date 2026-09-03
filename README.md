# Whitechain Bootstrap Files

Canonical public source of bootstrap and genesis artefacts for Whitechain networks.

The files here are what a node operator needs to start a Whitechain node from block 0, and what an external party needs to independently verify the chain configuration and the deployed L1 contract set.

## Networks

| Network | Directory | L2 chain ID | Settlement layer | Status |
| --- | --- | --- | --- | --- |
| Testnet | `testnet/` | 1874 | Ethereum Sepolia (11155111) | published |
| Mainnet | `mainnet/` | – | – | not yet published |

Each network directory is self-contained. Never mix files across networks – a genesis from one network will not match the rollup config of another.

## Files

| File | Purpose |
| --- | --- |
| `genesis.json` | L2 execution-layer genesis. Consumed by the execution client (`op-geth init`, or `op-reth --chain`) to build the block 0 state, including all predeploys and pre-funded allocations. |
| `rollup.json` | L2 consensus-layer (rollup) config. Consumed by `op-node`. Defines the L1 origin block, protocol activation times, sequencing windows and L1 system contract addresses. |
| `l1-addresses.json` | Addresses of all L1 contracts deployed for the chain – portal, bridges, messengers, system config, dispute game factory and their implementations. Use it to verify on-chain deployments on the settlement layer. |
| `intent.toml` | Deployment intent given to `op-deployer` – the human-authored input describing chain parameters, roles, fee recipients and the custom gas token. The source of truth for how the chain was configured. |
| `deploy-config.json` | Resolved OP Stack deploy config – the full parameter set the chain was deployed with, after intent defaults and overrides were merged. Published so the effective configuration can be read and diffed without running `op-deployer`. |
| `state.json` | `op-deployer` output state – the full record of the deployment, including the applied intent and every deployed contract address. `genesis.json`, `rollup.json`, `l1-addresses.json` and `deploy-config.json` are all derivable from it. |

Note: only `genesis.json` and `rollup.json` are read by a running node. The other four files exist so the chain configuration and the L1 deployment can be checked independently, and all four are regenerated deterministically from `state.json` (see below).

## Usage

Initialise an execution client. Either client works – the same `genesis.json` is accepted by both.

`op-geth`:

```bash
op-geth init --datadir=./datadir testnet/genesis.json
```

`op-reth`:

```bash
op-reth init --chain=testnet/genesis.json --datadir=./datadir
```

Note: `op-reth` takes the genesis file through `--chain` on every invocation, not only on init, so the same `--chain=testnet/genesis.json` must be passed to `op-reth node`. The explicit `init` step is optional – `op-reth node` initialises an empty datadir from `--chain` on first start.

Start the consensus client against the same network:

```bash
op-node \
  --rollup.config=testnet/rollup.json \
  --l1=<L1_RPC> \
  --l1.beacon=<L1_BEACON> \
  --l2=<L2_ENGINE_RPC> \
  --l2.jwt-secret=<JWT_FILE> ...
```

`<L2_ENGINE_RPC>` is the authenticated engine API of the execution client started above (`--authrpc.*` on both `op-geth` and `op-reth`), and `<JWT_FILE>` must be the same JWT secret that client was given.

Regenerate the derived artefacts from the deployer state and confirm they match the committed files:

```bash
op-deployer inspect genesis       --workdir ./testnet 1874 > /tmp/genesis.json
op-deployer inspect rollup        --workdir ./testnet 1874 > /tmp/rollup.json
op-deployer inspect deploy-config --workdir ./testnet 1874 > /tmp/deploy-config.json
op-deployer inspect l1            --workdir ./testnet 1874 > /tmp/l1-addresses.json

diff <(jq -S . testnet/genesis.json)       <(jq -S . /tmp/genesis.json)
diff <(jq -S . testnet/rollup.json)        <(jq -S . /tmp/rollup.json)
diff <(jq -S . testnet/deploy-config.json) <(jq -S . /tmp/deploy-config.json)
diff <(jq -S . testnet/l1-addresses.json)  <(jq -S . /tmp/l1-addresses.json)
```

Verify integrity of what you downloaded:

```bash
shasum -a 256 testnet/*
```

Expected values for the current testnet set:

```
1aa1680515f0118dec543342a6a6689c553b7e2e576a3014eb3975af8f738a40  testnet/deploy-config.json
7097a49f0ac0a7a9dc36a68c8e401018d0238a15730b08a2d1874efd2d422cf5  testnet/genesis.json
c48a33129a238f97af788dc9805894cd7761a58c20ac108d2a1daf5315eeaef7  testnet/intent.toml
0b84cc7b4f818846c40874cd9c0c3afc964978dfaa9e312959a8069997b405c3  testnet/l1-addresses.json
722deb00df45e276cff35d66c40d15cbbb3c5e42f1086077ad6beb55b8bb16b5  testnet/rollup.json
25aed7f30c4f8807a20a56a5202766e84c076bfb90924ef1a9bb0052adbfb793  testnet/state.json
```
