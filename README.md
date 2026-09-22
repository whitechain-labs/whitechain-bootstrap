# Whitechain Bootstrap Files

Canonical public source of bootstrap and genesis artefacts for Whitechain networks.

The files here are what a node operator needs to start a Whitechain node from block 0, and what an external party needs to independently verify the chain configuration and the deployed L1 contract set.

## Networks

| Network | Directory | L2 chain ID | Settlement layer | Status |
| --- | --- | --- | --- | --- |
| Testnet | `testnet/` | 1874 | Ethereum Sepolia (11155111) | published |
| Mainnet | `mainnet/` | 1875 | Ethereum (1) | not yet published |

Each network directory is self-contained. Never mix files across networks – a genesis from one network will not match the rollup config of another.

The mainnet set is published ahead of the launch for review. Its contract addresses and deployment hashes are not final and will change before the chain goes live, so do not treat the current mainnet files as canonical.

## Files

| File | Purpose |
| --- | --- |
| `genesis.json` | L2 execution-layer genesis. Consumed by the execution client (`op-geth init`, or `op-reth --chain`) to build the block 0 state, including all predeploys and pre-funded allocations. Mainnet ships it as `genesis.json.zip` (about 154 MB unpacked) – unzip it before use. |
| `rollup.json` | L2 consensus-layer (rollup) config. Consumed by `op-node`. Defines the L1 origin block, protocol activation times, sequencing windows and L1 system contract addresses. |
| `chain-info.json` | On-chain view of the deployed chain, read back from L1: the L1 proxy and implementation addresses the chain actually uses, the chain roles (system config owner, proxy admin owner, guardian, unsafe block signer, batch submitter, proposer, challenger) and the fault-proof configuration. Use it to check the committed artefacts against the live deployment. |
| `l1-addresses.json` | Deployer record of every L1 contract deployed for the chain – portal, bridges, messengers, system config, dispute game factory, superchain-level contracts and all implementations. Broader than `chain-info.json`: it lists what was deployed, while `chain-info.json` reports what the chain is wired to. |
| `l2-deploy-config.raw.json` | Flat L2 deploy config derived from `state.json`. The single-file view of every chain parameter: fee vaults and recipients, EIP-1559 parameters, hardfork activation offsets, custom gas token, batch inbox, dispute-game and withdrawal delays. Previously named `deploy-config.json` – the rename is a rename only, the content is unchanged. |
| `intent.toml` | Deployment intent given to `op-deployer` – the human-authored input describing chain parameters, roles, fee recipients and the custom gas token. The source of truth for how the chain was configured, and it records the `op-deployer` version used. |
| `state.json` | `op-deployer` output state – the full record of the deployment, including the applied intent and every deployed contract address. `genesis.json`, `rollup.json`, `l2-deploy-config.raw.json` and `l1-addresses.json` are all derivable from it. |

## Tooling

`chain-info.json` is produced by `op-fetcher`, which reads the state directly from L1.

Everything else is produced by `op-deployer`. The version differs per network and is recorded in each network's `intent.toml` and `state.json`:

| Network | `op-deployer` version |
| --- | --- |
| Mainnet | `v0.7.1-75254822-1779890299` |
| Testnet | `0.6.0-e8abf571-2026-04-02T10:18:56Z` |

## Usage

Mainnet only – unpack the genesis first:

```bash
unzip mainnet/genesis.json.zip -d mainnet/
```

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

## Verification

Regenerate the derived artefacts from the deployer state and confirm they match the committed files:

```bash
op-deployer inspect genesis       --workdir ./testnet 1874 > /tmp/genesis.json
op-deployer inspect rollup        --workdir ./testnet 1874 > /tmp/rollup.json
op-deployer inspect deploy-config --workdir ./testnet 1874 > /tmp/l2-deploy-config.raw.json
op-deployer inspect l1            --workdir ./testnet 1874 > /tmp/l1-addresses.json

diff <(jq -S . testnet/genesis.json) <(jq -S . /tmp/genesis.json)
diff <(jq -S . testnet/rollup.json)  <(jq -S . /tmp/rollup.json)
```

For mainnet, use `--workdir ./mainnet 1875`. Use the `op-deployer` version the network was deployed with (see Tooling) – other versions may produce a different output layout.

Re-fetch the on-chain view and confirm it matches the committed `chain-info.json`. The two contract addresses below are the testnet `SystemConfigProxy` and `L1StandardBridgeProxy`, taken from that same file:

```bash
op-fetcher fetch \
  --l1-rpc-url=<L1_RPC> \
  --system-config=0x9328ea869949f33c57b7b680b6edb58769e2181c \
  --l1-standard-bridge=0x0c50be539ab5d72d226038928f2eb25100899ded \
  --output-file=/tmp/chain-info.json

diff <(jq -S . testnet/chain-info.json) <(jq -S . /tmp/chain-info.json)
```

Verify integrity of what you downloaded:

```bash
shasum -a 256 testnet/*.json testnet/*.toml
```

Expected values for the current testnet set:

```
865825f767651a179a466d2a009b02be598111de497a868074f18132dabb34f1  testnet/chain-info.json
7097a49f0ac0a7a9dc36a68c8e401018d0238a15730b08a2d1874efd2d422cf5  testnet/genesis.json
0b84cc7b4f818846c40874cd9c0c3afc964978dfaa9e312959a8069997b405c3  testnet/l1-addresses.json
1aa1680515f0118dec543342a6a6689c553b7e2e576a3014eb3975af8f738a40  testnet/l2-deploy-config.raw.json
722deb00df45e276cff35d66c40d15cbbb3c5e42f1086077ad6beb55b8bb16b5  testnet/rollup.json
25aed7f30c4f8807a20a56a5202766e84c076bfb90924ef1a9bb0052adbfb793  testnet/state.json
c48a33129a238f97af788dc9805894cd7761a58c20ac108d2a1daf5315eeaef7  testnet/intent.toml
```

Mainnet checksums will be published together with the final mainnet set.
