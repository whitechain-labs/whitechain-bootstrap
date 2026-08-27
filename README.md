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
| `genesis.json` | L2 execution-layer genesis. Consumed by `op-geth init` to build the block 0 state, including all predeploys and pre-funded allocations. |
| `rollup.json` | L2 consensus-layer (rollup) config. Consumed by `op-node`. Defines the L1 origin block, protocol activation times, sequencing windows and L1 system contract addresses. |
| `l1-addresses.json` | Addresses of all L1 contracts deployed for the chain – portal, bridges, messengers, system config, dispute game factory and their implementations. Use it to verify on-chain deployments on the settlement layer. |
| `intent.toml` | Deployment intent given to `op-deployer` – the human-authored input describing chain parameters, roles, fee recipients and the custom gas token. The source of truth for how the chain was configured. |
| `state.json` | `op-deployer` output state – the full record of the deployment, including the applied intent and every deployed contract address. `genesis.json`, `rollup.json` and the deploy config are all derivable from it. |

Note: `deploy-config` is not stored as a separate file, it is contained in `state.json` and can be regenerated deterministically (see below).

## Usage

Initialise an execution client:

```bash
op-geth init --datadir=./datadir testnet/genesis.json
```

Start the consensus client against the same network:

```bash
op-node --rollup.config=testnet/rollup.json --l1=<L1_RPC> --l1.beacon=<L1_BEACON> ...
```

Regenerate the derived artefacts from the deployer state and confirm they match the committed files:

```bash
op-deployer inspect genesis       --workdir ./testnet 1874 > /tmp/genesis.json
op-deployer inspect rollup        --workdir ./testnet 1874 > /tmp/rollup.json
op-deployer inspect deploy-config --workdir ./testnet 1874 > /tmp/deploy-config.json
op-deployer inspect l1            --workdir ./testnet 1874 > /tmp/l1-addresses.json

diff <(jq -S . testnet/genesis.json) <(jq -S . /tmp/genesis.json)
diff <(jq -S . testnet/rollup.json)  <(jq -S . /tmp/rollup.json)
```

Verify integrity of what you downloaded:

```bash
shasum -a 256 testnet/*
```

Expected values for the current testnet set:

```
7097a49f0ac0a7a9dc36a68c8e401018d0238a15730b08a2d1874efd2d422cf5  testnet/genesis.json
c48a33129a238f97af788dc9805894cd7761a58c20ac108d2a1daf5315eeaef7  testnet/intent.toml
0b84cc7b4f818846c40874cd9c0c3afc964978dfaa9e312959a8069997b405c3  testnet/l1-addresses.json
722deb00df45e276cff35d66c40d15cbbb3c5e42f1086077ad6beb55b8bb16b5  testnet/rollup.json
25aed7f30c4f8807a20a56a5202766e84c076bfb90924ef1a9bb0052adbfb793  testnet/state.json
```
