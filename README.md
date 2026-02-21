# Cardano SPO Setup Guide (Preprod Testnet)

インフラエンジニアが VirtualBox 上の Ubuntu 1台で、リレーノードとブロックプロデューサー(BP)ノードを論理分離して構築したガイド。

---

## 1. システム構成
- **Node Version:** 10.1.4 (Conway対応)
- **Network:** Preprod Testnet
- **Instance:** Ubuntu 22.04 LTS (VirtualBox)
- **Configuration:** 1 VM / 2 Processes (Logical Separation)

### ネットワーク図
```text
[Internet] <--> [Router (NAT)] <--> [Windows Host (NAT PortForward 2222->22)]
                                          |
                                  [VirtualBox Ubuntu VM]
                                  |-- Relay Node (Port 3001)
                                  `-- BP Node (Port 3002) <--> Local P2P
