# Cardano SPO Setup Guide (Preprod Testnet)

インフラエンジニアが VirtualBox 上の Ubuntu 1台で、リレーノードとブロックプロデューサー(BP)ノードを論理分離して構築したガイド。

---

## 1. システム構成
- **Node Version:** 10.1.4 (Conway対応)
- **Network:** Preprod Testnet
- **Instance:** Ubuntu 22.04 LTS (VirtualBox)
- **Configuration:** 1 VM / 2 Processes (Logical Separation)

### ネットワーク図
~~~text
[Internet] <--> [Router (NAT)] <--> [Windows Host (NAT PortForward 2222->22)]
                                          |
                                  [VirtualBox Ubuntu VM]
                                  |-- Relay Node (Port 3001)
                                  `-- BP Node (Port 3002) <--- Local P2P ---> Relay
~~~

---

## 2. ディレクトリ構成
インフラの役割ごとにディレクトリを分離し、管理性とセキュリティを確保。
~~~bash
/home/<user_name>/cardano-node/
├── relay/      # リレーノード用（DB、設定ファイル、ソケット）
├── bp/         # BPノード用（DB、設定ファイル、ホットキー、証明書）
├── keys/       # SPO秘密鍵保管庫（本来はオフライン環境で管理）
└── scripts/    # 運用支援スクリプト（監視、スケジュール確認用）
~~~

---

## 3. ネットワーク設定 (P2P Topology)

### Relay Node (`relay/config/topology.json`)
~~~json
{
  "localRoots": [{
      "accessPoints": [{"address": "127.0.0.1", "port": 3002}],
      "advertise": false, "trustable": true, "valency": 1
  }],
  "publicRoots": [{
      "accessPoints": [{"address": "preprod-node.world.dev.cardano.org", "port": 30000}],
      "advertise": false
  }],
  "useLedgerAfterSlot": 0
}
~~~

### BP Node (`bp/config/topology.json`)
~~~json
{
  "localRoots": [{
      "accessPoints": [{"address": "127.0.0.1", "port": 3001}],
      "advertise": false, "trustable": true, "valency": 1
  }],
  "publicRoots": [],
  "useLedgerAfterSlot": 0
}
~~~

---

## 4. サービス定義 (Systemd)

### Relay Service (`/etc/systemd/system/cardano-relay.service`)
~~~ini
[Service]
WorkingDirectory=/home/<user_name>/cardano-node/relay
ExecStart=/usr/local/bin/cardano-node run \
  --config /home/<user_name>/cardano-node/relay/config/config.json \
  --topology /home/<user_name>/cardano-node/relay/config/topology.json \
  --database-path /home/<user_name>/cardano-node/relay/db \
  --socket-path /home/<user_name>/cardano-node/relay/node.socket \
  --host-addr 0.0.0.0 \
  --port 3001
LimitNOFILE=131072
Restart=always
~~~

### BP Service (`/etc/systemd/system/cardano-bp.service`)
~~~ini
[Service]
WorkingDirectory=/home/<user_name>/cardano-node/bp
ExecStart=/usr/local/bin/cardano-node run \
  --config /home/<user_name>/cardano-node/bp/config.json \
  --topology /home/<user_name>/cardano-node/bp/config/topology.json \
  --database-path /home/<user_name>/cardano-node/bp/db \
  --socket-path /home/<user_name>/cardano-node/bp/node.socket \
  --host-addr 127.0.0.1 \
  --port 3002 \
  --shelley-kes-key /home/<user_name>/cardano-node/bp/kes.skey \
  --shelley-vrf-key /home/<user_name>/cardano-node/bp/vrf.skey \
  --shelley-operational-certificate /home/<user_name>/cardano-node/bp/node.cert
LimitNOFILE=131072
Restart=always
~~~

---

## 5. オンチェーン登録コマンド (v10.1.4)

### ステークアドレス登録
~~~bash
cardano-cli latest stake-address registration-certificate \
    --stake-verification-key-file stake.vkey \
    --key-reg-deposit-amt 2000000 \
    --out-file stake.cert
~~~

### プール登録
~~~bash
cardano-cli latest stake-pool registration-certificate \
    --cold-verification-key-file node.vkey \
    --vrf-verification-key-file vrf.vkey \
    --pool-pledge 4500000000 \
    --pool-cost 170000000 \
    --pool-margin 0.02 \
    --reward-account-verification-key-file stake.vkey \
    --pool-owner-stake-verification-key-file stake.vkey \
    --testnet-magic 1 \
    --pool-relay-ipv4 <public_ip> \
    --pool-relay-port 3001 \
    --metadata-url <shortened_url> \
    --metadata-hash $(cat poolMetaData.hash) \
    --out-file pool.cert
~~~

---

## 6. 保守運用の鉄則
1. **DB保護:** シャットダウン時は `systemctl stop` を実行し、データのFlush（数秒〜数十秒）を待つ。
2. **Pledge維持:** 宣言したPledge額（4500 ADA）を下回らないよう残高を管理。
3. **ディスク管理:** `TraceMempool: false` 設定と `SystemMaxUse=500M` でログ肥大化を防止。
4. **回線安定化:** SPO運用には有線LAN接続を推奨。

---
**Ticker:** hrdep / **Live Stake:** ~20k ADA
