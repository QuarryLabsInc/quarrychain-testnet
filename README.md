# QuarryChain Testnet: run a validator

Everything an operator needs to join the QuarryChain testnet as a validator:
the `quarrychaind` binary, the genesis file, and the steps to run a node and
register it.

Register at **https://governance.quarrychain.network/onboarding** once your node is synced.

## Network

| | |
| --- | --- |
| Chain ID (Cosmos) | `quarrychain_13133-1` |
| EVM chain ID | 13133 |
| Binary | `quarrychaind`, commit `a9913f1` (Cosmos EVM v0.7.3, CometBFT) |
| Genesis SHA-256 | `5a71a4a8196a45e4abda1e4f5a7a0aec8e257ea5a21fad1889619d6f0ea15b9b` |
| Peer | `e33351b981f33af71e748d964ff5c5c79e95ea17@74.208.55.48:30056` |
| Minimum gas price | `10000000000aqry` (10 gwei) |
| Public EVM RPC | https://rpc.testnet.quarrychain.network |
| Explorer | https://explorer.testnet.quarrychain.network |
| Self-bond | 10,000 QRY minimum, from your EVM wallet |

Downloads are on the [`testnet-a9913f1` release](../../releases/tag/testnet-a9913f1):
`quarrychaind-linux-amd64`, `genesis.json` and `SHA256SUMS`.

## Server

- Linux x86_64, glibc 2.36 or newer (Debian 12, Ubuntu 22.04 or later).
- 2+ vCPU, 4 GB+ RAM. A validator uses about 2.5 GB.
- SSD, 200 GB to start. History grows about 0.35 GB a day.
- Inbound TCP **26656** (p2p) open. JSON-RPC **8545** open for the portal's
  connection check; plain `http://` is fine. Keep CometBFT RPC (26657) on
  localhost.

## 1. Download, verify, install

```bash
R=https://github.com/QuarryLabsInc/quarrychain-testnet/releases/download/testnet-a9913f1
curl -fsSLO $R/quarrychaind-linux-amd64 -fsSLO $R/genesis.json -fsSLO $R/SHA256SUMS
sha256sum -c SHA256SUMS          # both lines must say OK
sudo install -m 755 quarrychaind-linux-amd64 /usr/local/bin/quarrychaind
quarrychaind version             # a9913f1
```

## 2. Initialise

```bash
quarrychaind init <your-moniker> --chain-id quarrychain_13133-1
cp genesis.json ~/.quarrychaind/config/genesis.json
```

`init` creates `~/.quarrychaind/config/priv_validator_key.json`, your
**consensus key**. Back it up offline. It must run in exactly **one** place:
two nodes signing with the same key is a double-sign, slashed 5% and jailed
permanently.

## 3. Configure

```bash
C=~/.quarrychaind/config
sed -i 's|^persistent_peers = .*|persistent_peers = "e33351b981f33af71e748d964ff5c5c79e95ea17@74.208.55.48:30056"|' $C/config.toml
sed -i 's|^external_address = .*|external_address = "<your-public-ip>:26656"|' $C/config.toml
sed -i 's|^minimum-gas-prices = .*|minimum-gas-prices = "10000000000aqry"|' $C/app.toml
sed -i '/^\[json-rpc\]/,/^\[/{s|^enable = .*|enable = true|;s|^address = .*|address = "0.0.0.0:8545"|}' $C/app.toml
```

This sets the peer, your public address, the gas price, and turns on EVM
JSON-RPC (off by default) on all interfaces.

## 4. Run under systemd

`/etc/systemd/system/quarrychaind.service`:

```ini
[Unit]
Description=QuarryChain testnet node
After=network-online.target
Wants=network-online.target

[Service]
User=<your-user>
ExecStart=/usr/local/bin/quarrychaind start --chain-id quarrychain_13133-1
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now quarrychaind
journalctl -u quarrychaind -f
```

## 5. Wait for the sync

The node replays the chain from block 1. Expect several hours: about
1,400 blocks a minute on an 8-vCPU machine (tested 2026-11-27), so roughly
7 hours at the current height of ~590,000; longer on smaller servers.

```bash
curl -s localhost:26657/status | jq '.result.sync_info | {latest_block_height, catching_up}'
```

Register only once `catching_up` is `false`. A validator whose node is behind
misses blocks and gets jailed.

## 6. Register on the portal

```bash
quarrychaind comet show-node-id      # Peer ID
quarrychaind comet show-validator    # consensus key: paste the whole JSON
```

1. Fund your EVM wallet (for example MetaMask, network: RPC
   `https://rpc.testnet.quarrychain.network`, chain ID 13133, currency QRY)
   with **10,100+ QRY**: 10,000 self-bond plus gas.
2. Open https://governance.quarrychain.network/onboarding and connect the wallet.
3. Step 1: your validator's name, website and description (stored on-chain).
4. Step 2: pick your host. **RPC endpoint** is your node's JSON-RPC, for example
   `http://<your-public-ip>:8545`. Click **Check Connection**: it must report
   chain 13133 and synced. Then paste the Peer ID and the consensus key.
5. Step 3: sign the `create-validator` transaction (10,000 QRY self-bond),
   then sign the profile message.

Each wallet and each consensus key can register only one validator.

## 7. Operate

- **Signing:** `quarrychaind query slashing signing-info "$(quarrychaind comet show-validator)"`.
  `missed_blocks_counter` should stay near 0.
- **Downtime:** signing less than 5% of a 10,000-block window (about 8–9 hours
  offline) jails the validator. Testnet downtime slashing is 0, but a jailed
  validator leaves the active set and earns nothing until it unjails.
- **Moving servers:** stop the old node and make sure it can't restart
  **before** starting the new one with the same key.
- **Leaving:** undelegate your self-bond through the staking precompile
  (`0x0000000000000000000000000000000000000800`, `undelegate`). It returns
  after the 1-day unbonding period.

## How the binary is built

From `quarrychain-node` commit `a9913f18b0b15abdefb9bf18cc4ebe8c2e93f249`
(the commit the network's validators run), with Go 1.25.9 in the
`golang:1.25.9-bookworm` image:
`make build VERSION=a9913f1 COMMIT=a9913f18b0b15abdefb9bf18cc4ebe8c2e93f249`.
`SHA256SUMS` on the release is the reference for the downloaded files.
