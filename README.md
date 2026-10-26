# Whitechain Bootstrap Files

Canonical public source of bootstrap and genesis artefacts for Whitechain networks.

The files here are what a node operator needs to start a Whitechain node from the L2 genesis block (block 0 on testnet, block `50961390` on mainnet), and what an external party needs to independently verify the chain configuration and the deployed L1 contract set.

## Networks

| Network | Directory | L2 chain ID | Settlement layer | Status |
| --- | --- | --- | --- | --- |
| Testnet | `testnet/` | 1874 | Ethereum Sepolia (11155111) | published |
| Mainnet | `mainnet/` | 1875 | Ethereum (1) | published |

Each network directory is self-contained. Never mix files across networks – a genesis from one network will not match the rollup config of another.

The mainnet set is final. Its contract addresses and deployment hashes match the live chain, and the files in `mainnet/` are canonical.

## Files

| File | Purpose |
| --- | --- |
| `genesis.json` | L2 execution-layer genesis. Consumed by the execution client (`op-geth init`, or `op-reth --chain`) to build the L2 genesis state, including all predeploys and pre-funded allocations. On mainnet the genesis block is `50961390` and its state includes the account state carried over from L1 Whitechain. Mainnet ships it as `genesis.json.zip` (about 154 MB unpacked) – unzip it before use. |
| `rollup.json` | L2 consensus-layer (rollup) config. Consumed by `op-node`. Defines the L1 origin block, protocol activation times, sequencing windows and L1 system contract addresses. |
| `chain-info.json` | On-chain view of the deployed chain, read back from L1: the L1 proxy and implementation addresses the chain actually uses, the chain roles (system config owner, proxy admin owner, guardian, unsafe block signer, batch submitter, proposer, challenger) and the fault-proof configuration. Use it to check the committed artefacts against the live deployment. |
| `l1-addresses.json` | Deployer record of every L1 contract deployed for the chain – portal, bridges, messengers, system config, dispute game factory, superchain-level contracts and all implementations. Broader than `chain-info.json`: it lists what was deployed, while `chain-info.json` reports what the chain is wired to. |
| `l2-deploy-config.raw.json` | Flat L2 deploy config derived from `state.json`. The single-file view of every chain parameter: fee vaults and recipients, EIP-1559 parameters, hardfork activation offsets, custom gas token, batch inbox, dispute-game and withdrawal delays. Previously named `deploy-config.json` – the rename is a rename only, the content is unchanged. Mainnet ships it as `l2-deploy-config.json`. |
| `intent.toml` | Deployment intent given to `op-deployer` – the human-authored input describing chain parameters, roles, fee recipients and the custom gas token. The source of truth for how the chain was configured, and it records the `op-deployer` version used. |
| `state.json` | `op-deployer` output state – the full record of the deployment, including the applied intent and every deployed contract address. `genesis.json`, `rollup.json`, `l2-deploy-config.raw.json` and `l1-addresses.json` are all derivable from it. |
| `state_dump_50961389.json` | Mainnet only. State dump of the L1 Whitechain network taken at its last block (`50961389`). Mainnet L2 continues from this state, so this file is the record of the account state carried over from the L1 Whitechain network. Shipped as `state_dump_50961389.json.zip` (about 131 MB unpacked) – unzip it before use. |

## Mainnet migration from L1 Whitechain

Mainnet continues the existing L1 Whitechain network. The L1 Whitechain state was dumped at its last block and carried over into the L2 chain.

L1 Whitechain has stopped producing blocks at block `50961389` and now works in read-only mode only: historical data stays queryable, but no new transactions are accepted. All new activity happens on L2 Whitechain, starting from block `50961390`.

| Item | Value |
| --- | --- |
| Last L1 Whitechain block | `50961389` |
| First L2 Whitechain block | `50961390` |
| L1 Whitechain state dump | `mainnet/state_dump_50961389.json.zip` |
| Archive SHA-256 | `bf3daa9fceed7209d57d959ab97ba9c3fe06f3c4868c22452d2e7aa6c1c74a52` |
| Unpacked `state_dump_50961389.json` SHA-256 | `13ced0e6ba78fae1cfe1f9e9a376dfb85c08f13a593ffabd8e00a60a8b5286b7` |

Unpack and verify the state dump:

```bash
shasum -a 256 mainnet/state_dump_50961389.json.zip
unzip mainnet/state_dump_50961389.json.zip -d mainnet/
shasum -a 256 mainnet/state_dump_50961389.json
```

```
bf3daa9fceed7209d57d959ab97ba9c3fe06f3c4868c22452d2e7aa6c1c74a52  mainnet/state_dump_50961389.json.zip
13ced0e6ba78fae1cfe1f9e9a376dfb85c08f13a593ffabd8e00a60a8b5286b7  mainnet/state_dump_50961389.json
```

## Tooling

`chain-info.json` is produced by `op-fetcher`, which reads the state directly from L1.

Everything else is produced by `op-deployer`. The version differs per network and is recorded in each network's `intent.toml` and `state.json`:

| Network | `op-deployer` version |
| --- | --- |
| Mainnet | `v0.8.1-319e3da9-1791906182` |
| Testnet | `0.6.0-e8abf571-2026-04-02T10:18:56Z` |

## Usage

The examples below use mainnet. For testnet, replace `mainnet/` with `testnet/` in every path – the testnet genesis is shipped unzipped, so skip the unpack step.

Unpack the mainnet genesis first:

```bash
unzip mainnet/genesis.json.zip -d mainnet/
```

Initialise an execution client. Either client works – the same `genesis.json` is accepted by both.

`op-geth`:

```bash
op-geth init --datadir=./datadir mainnet/genesis.json
```

`op-reth`:

```bash
op-reth init --chain=mainnet/genesis.json --datadir=./datadir
```

Note: `op-reth` takes the genesis file through `--chain` on every invocation, not only on init, so the same `--chain=mainnet/genesis.json` must be passed to `op-reth node`. The explicit `init` step is optional – `op-reth node` initialises an empty datadir from `--chain` on first start.

Start the consensus client against the same network:

```bash
op-node \
  --rollup.config=mainnet/rollup.json \
  --l1=<L1_RPC> \
  --l1.beacon=<L1_BEACON> \
  --l2=<L2_ENGINE_RPC> \
  --l2.jwt-secret=<JWT_FILE> ...
```

`<L1_RPC>` and `<L1_BEACON>` must point to the network's settlement layer – Ethereum mainnet for mainnet, Ethereum Sepolia for testnet. `<L2_ENGINE_RPC>` is the authenticated engine API of the execution client started above (`--authrpc.*` on both `op-geth` and `op-reth`), and `<JWT_FILE>` must be the same JWT secret that client was given.

## Verification

### Regenerate derived artefacts

Regenerate the derived artefacts from the deployer state and confirm they match the committed files. Use the `op-deployer` version the network was deployed with (see Tooling) – other versions may produce a different output layout.

Mainnet:

```bash
unzip mainnet/genesis.json.zip -d mainnet/

op-deployer inspect genesis       --workdir ./mainnet 1875 > /tmp/genesis.json
op-deployer inspect rollup        --workdir ./mainnet 1875 > /tmp/rollup.json
op-deployer inspect deploy-config --workdir ./mainnet 1875 > /tmp/l2-deploy-config.json
op-deployer inspect l1            --workdir ./mainnet 1875 > /tmp/l1-addresses.json

diff <(jq -S . mainnet/genesis.json)           <(jq -S . /tmp/genesis.json)
diff <(jq -S . mainnet/rollup.json)            <(jq -S . /tmp/rollup.json)
diff <(jq -S . mainnet/l2-deploy-config.json)  <(jq -S . /tmp/l2-deploy-config.json)
diff <(jq -S . mainnet/l1-addresses.json)      <(jq -S . /tmp/l1-addresses.json)
```

Testnet:

```bash
op-deployer inspect genesis       --workdir ./testnet 1874 > /tmp/genesis.json
op-deployer inspect rollup        --workdir ./testnet 1874 > /tmp/rollup.json
op-deployer inspect deploy-config --workdir ./testnet 1874 > /tmp/l2-deploy-config.raw.json
op-deployer inspect l1            --workdir ./testnet 1874 > /tmp/l1-addresses.json

diff <(jq -S . testnet/genesis.json) <(jq -S . /tmp/genesis.json)
diff <(jq -S . testnet/rollup.json)  <(jq -S . /tmp/rollup.json)
```

### Re-fetch the on-chain view

Re-fetch the on-chain view and confirm it matches the committed `chain-info.json`. The two contract addresses in each command are that network's `SystemConfigProxy` and `L1StandardBridgeProxy`, taken from its own `chain-info.json`.

Mainnet (`<L1_RPC>` is an Ethereum mainnet RPC):

```bash
op-fetcher fetch \
  --l1-rpc-url=<L1_RPC> \
  --system-config=0x1d761826ad3098d10f6ddfbd143957aaf2b6e50c \
  --l1-standard-bridge=0x7e954f20f24b7988ea966c4e7880fe6d21dcc2d9 \
  --output-file=/tmp/chain-info.json

diff <(jq -S . mainnet/chain-info.json) <(jq -S . /tmp/chain-info.json)
```

Testnet (`<L1_RPC>` is an Ethereum Sepolia RPC):

```bash
op-fetcher fetch \
  --l1-rpc-url=<L1_RPC> \
  --system-config=0x9328ea869949f33c57b7b680b6edb58769e2181c \
  --l1-standard-bridge=0x0c50be539ab5d72d226038928f2eb25100899ded \
  --output-file=/tmp/chain-info.json

diff <(jq -S . testnet/chain-info.json) <(jq -S . /tmp/chain-info.json)
```

### Checksums

Verify integrity of what you downloaded.

Mainnet:

```bash
shasum -a 256 mainnet/*.json mainnet/*.toml mainnet/*.zip
```

Expected values for the current mainnet set. The two `genesis.json` and `state_dump_50961389.json` lines apply after unzipping:

```
5453c24e23cd23b6798607e27e27197af506d6e518eb42a911457193603815a9  mainnet/chain-info.json
06912e56e12c5910b5dbfc3a81dbb72a6f80fb69a246d22fda5bef1bb4ddb295  mainnet/genesis.json
4fc87d6d092f6dbe1e7be37a78bbfc1bafb4c8d38c17c01f75326bf9875f6df9  mainnet/l1-addresses.json
8bb0524b0f9fecdb5c23ab6eef20e9f8f0a2f1f5dc55f27ff5f02f8eb6dc80bd  mainnet/l2-deploy-config.json
116c8cf0e4c4ee6afa7a30de12e0250f1a6805ac33875d2a3de81721c705f623  mainnet/rollup.json
8e31879a1db68fa1ff249caafd342e3e1602714be24b38055cf0c62d2c224704  mainnet/state.json
13ced0e6ba78fae1cfe1f9e9a376dfb85c08f13a593ffabd8e00a60a8b5286b7  mainnet/state_dump_50961389.json
2ad56e4fceac00275cd918c217b1bf57e1e015f560b8f0fe44a0b7a559b5af04  mainnet/intent.toml
b0db408c654eb01e3813afe1ff790b1822fbfd9d7cf443181f2fb1063311c80a  mainnet/genesis.json.zip
bf3daa9fceed7209d57d959ab97ba9c3fe06f3c4868c22452d2e7aa6c1c74a52  mainnet/state_dump_50961389.json.zip
```

Testnet:

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
