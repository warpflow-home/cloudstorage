# 🐳 Docker Swarm HA クラスター構築ガイド

## Ubuntu 24.04 LTS | 4ノード | GlusterFS | Keepalived(VIP) | macvlan/overlay

---

## 📋 目次

1. [アーキテクチャ概要](https://www.google.com/search?q=%23%E3%82%A2%E3%83%BC%E3%82%AD%E3%83%86%E3%82%AF%E3%83%81%E3%83%A3%E6%A6%82%E8%A6%81)
2. [フェーズ1: 全ノード共通設定](https://www.google.com/search?q=%23%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA1-%E5%85%A8%E3%83%8E%E3%83%BC%E3%83%89%E5%85%B1%E9%80%9A%E8%A8%AD%E5%AE%9A)
3. [フェーズ2: GlusterFS 分散ストレージ](https://www.google.com/search?q=%23%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA2-glusterfs-%E5%88%86%E6%95%A3%E3%82%B9%E3%83%88%E3%83%AC%E3%83%BC%E3%82%B8)
4. [フェーズ3: Docker Swarm クラスター構築](https://www.google.com/search?q=%23%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA3-docker-swarm-%E3%82%AF%E3%83%A9%E3%82%B9%E3%82%BF%E3%83%BC%E6%A7%8B%E7%AF%89)
5. [フェーズ4: ネットワーク設定 (macvlan + VIP + overlay)](https://www.google.com/search?q=%23%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA4-%E3%83%8D%E3%83%83%E3%83%88%E3%83%AF%E3%83%BC%E3%82%AF%E8%A8%AD%E5%AE%9A-macvlan--vip--overlay)
6. [フェーズ5: Portainer デプロイ (VIP経由)](https://www.google.com/search?q=%23%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA5-portainer-%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4-vip%E7%B5%8C%E7%94%B1)
7. [フェーズ6: Asterisk デプロイ (macvlan)](https://www.google.com/search?q=%23%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA6-asterisk-%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4-macvlan)
8. [フェーズ7: その他サービスのスタック例 (VIP経由)](https://www.google.com/search?q=%23%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA7-%E3%81%9D%E3%81%AE%E4%BB%96%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%81%AE%E3%82%B9%E3%82%BF%E3%83%83%E3%82%AF%E4%BE%8B-vip%E7%B5%8C%E7%94%B1)
9. [フェーズ8: メンテナンス & トラブルシューティング](https://www.google.com/search?q=%23%E3%83%95%E3%82%A7%E3%83%BC%E3%82%BA8-%E3%83%A1%E3%83%B3%E3%83%86%E3%83%8A%E3%83%B3%E3%82%B9--%E3%83%88%E3%83%A9%E3%83%96%E3%83%AB%E3%82%B7%E3%83%A5%E3%83%BC%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0)
10. [デプロイチェックリスト](https://www.google.com/search?q=%23%E3%83%87%E3%83%97%E3%83%AD%E3%82%A4%E3%83%81%E3%82%A7%E3%83%83%E3%82%AF%E3%83%AA%E3%82%B9%E3%83%88)

---

## アーキテクチャ概要

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                  物理ネットワーク: 192.168.1.0/24                          │
│                  ゲートウェイ:    192.168.1.1                              │
└──────────┬──────────────┬──────────────┬──────────────┬───────────────────┘
           │              │              │              │
  ┌────────▼──────┐ ┌─────▼──────┐ ┌────▼──────┐ ┌────▼──────────┐
  │node01(mgr+wkr)│ │node02(mgr) │ │node03(mgr)│ │node04(mgr+wkr)│
  │192.168.1.241  │ │192.168.1.242│ │192.168.1.243│ │192.168.1.244 │
  │GlusterFS brick│ │GlusterFS   │ │GlusterFS  │ │GlusterFS brick│
  └───────────────┘ └────────────┘ └───────────┘ └───────────────┘
           │              │              │              │
           ├──────────────┴──────────────┴──────────────┤
           │   Keepalived 仮想IP (VIP): 192.168.1.240    │
           │   (全ノードで共有。アクティブノードが応答)      │
           └──────────────────────┬─────────────────────┘
                                  │
  ┌───────────────────────────────▼───────────────────────────────┐
  │              Docker Overlay Network (swarm-overlay)           │
  │              サブネット: 10.10.0.0/16                          │
  │              用途: Portainer(9443), DB, Web等の一般サービス     │
  └───────────────────────────────────────────────────────────────┘
                                  
  ┌───────────────────────────────▼───────────────────────────────┐
  │              macvlan Network (物理NICにブリッジ)                │
  │              Asterisk: 192.168.1.200  ← 実LAN IP(SIP/RTP用)   │
  └───────────────────────────────────────────────────────────────┘


```

### 設計のポイント

| 設計 | 理由 |
| --- | --- |
| 全ノード = マネージャー + ワーカー | 4台中1台が落ちても3/4でクォーラム維持 |
| GlusterFS Replica 3 | 1ノード障害でもデータ完全保持 |
| Keepalived VIP (192.168.1.240) | 汎用サービスへのアクセス窓口を一つに統合。ノード障害時も自動フェイルオーバー。 |
| Asterisk に macvlan (192.168.1.200) | SIP/RTPはNATで壊れるため実IPが必須。VIPとは完全に分離。 |

### Raft クォーラム（重要）

```text
4マネージャー構成:
  ✅ node01 が落ちても → node02 + node03 + node04 で新リーダー選出
  ✅ node02 が落ちても → node01 + node03 + node04 でクォーラム維持
  ❌ 2台同時に落ちると → クォーラム失敗（クラスター停止）

💡 4台構成（偶数）は奇数構成より障害耐性が向上するわけではない
   4台中2台が落ちるとクォーラム失敗（3台構成も同じ）
   5台にして初めて2台同時障害に対応できる


```

---

## フェーズ1: 全ノード共通設定

> ⚠️ **以下の操作は node01・node02・node03・node04 の全てで実行してください**

### IPアドレス計画

```text
node01: 192.168.1.241   hostname: node01
node02: 192.168.1.242   hostname: node02
node03: 192.168.1.243   hostname: node03
node04: 192.168.1.244   hostname: node04

VIP (Keepalived):       192.168.1.240     (汎用サービス窓口、ポートで振り分け)
Asterisk (macvlan):     192.168.1.200     (直接LAN接続、SIP/RTP用)
Overlay サブネット:     10.10.0.0/16      (サービス間通信用)


```

### 1-1. ホスト名の設定

```bash
# 各ノードで対応するホスト名を設定
sudo hostnamectl set-hostname node01 # または node02, node03, node04


```

### 1-2. /etc/hosts の設定（全ノード共通）

```bash
sudo tee -a /etc/hosts <<'EOF'

# Docker Swarm Cluster
192.168.1.241  node01
192.168.1.242  node02
192.168.1.243  node03
192.168.1.244  node04
EOF


```

### 1-3. 必要パッケージとカーネルモジュールの設定

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  software-properties-common \
  nfs-common \
  glusterfs-client \
  keepalived \
  htop \
  net-tools \
  ufw

# カーネルモジュールのロード
sudo modprobe overlay
sudo modprobe br_netfilter
sudo modprobe dm_thin_pool
sudo modprobe 8021q

# 永続化（再起動後も有効）
sudo tee /etc/modules-load.d/cluster.conf <<'EOF'
overlay
br_netfilter
dm_thin_pool
8021q
EOF

# Dockerネットワーキングに必要なsysctl設定
sudo tee /etc/sysctl.d/99-docker-swarm.conf <<'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
# Asterisk の音声通信用にバッファを増やす
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
EOF

sudo sysctl --system


```

### 1-4. ファイアウォール設定

```bash
# Docker Swarm 必須ポート
sudo ufw allow 2376/tcp  comment 'Docker TLS'
sudo ufw allow 2377/tcp  comment 'Docker Swarm管理'
sudo ufw allow 7946/tcp  comment 'Swarmノード間通信'
sudo ufw allow 7946/udp  comment 'Swarmノード間通信'
sudo ufw allow 4789/udp  comment 'Docker overlay VXLAN'

# GlusterFS ポート
sudo ufw allow from 192.168.1.0/24 to any port 24007    comment 'GlusterFS daemon'
sudo ufw allow from 192.168.1.0/24 to any port 24008    comment 'GlusterFS管理'
sudo ufw allow from 192.168.1.0/24 to any port 49152:49251 proto tcp comment 'GlusterFS bricks'

# VRRP (Keepalived用)
sudo ufw allow in to 224.0.0.18 comment 'VRRP'

# Portainer ポート
sudo ufw allow 9000/tcp  comment 'Portainer HTTP'
sudo ufw allow 9443/tcp  comment 'Portainer HTTPS'
sudo ufw allow 8000/tcp  comment 'Portainer edge tunnel'
sudo ufw allow 9001/tcp  comment 'Portainer agent'

# Asterisk / SIP / RTP ポート
sudo ufw allow 5060/udp  comment 'SIP'
sudo ufw allow 5060/tcp  comment 'SIP TCP'
sudo ufw allow 5061/tcp  comment 'SIP TLS'
sudo ufw allow 10000:20000/udp comment 'RTP メディア'

# クラスター内ネットワーク全許可
sudo ufw allow from 192.168.1.0/24

sudo ufw --force enable
sudo ufw status verbose


```

### 1-5. Docker Engine のインストール

```bash
# 古いバージョンの削除
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Docker 公式GPGキーの追加
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Dockerリポジトリの追加
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

# 有効化と起動
sudo systemctl enable docker
sudo systemctl start docker

# sudoなしで実行
sudo usermod -aG docker $USER


```

---

## フェーズ2: GlusterFS 分散ストレージ

> **なぜ GlusterFS?** Docker Swarm にはネイティブの分散ストレージがありません。サービスが別ノードに移動した際、データが既にそこに存在している必要があります。GlusterFS Replica 3 は全3ノードに同一データを保持するため、どのノードからもデータにアクセスできます。

### 2-1. GlusterFS サーバーのインストール（全ノード）

```bash
# Ubuntu 24.04 Noble 向け GlusterFS 10 PPA
sudo add-apt-repository -y ppa:gluster/glusterfs-10
sudo apt update
sudo apt install -y glusterfs-server glusterfs-client

sudo systemctl enable glusterd
sudo systemctl start glusterd


```

### 2-2. ストレージブリックの準備（全ノード）

```bash
# 専用ディスクがある場合（推奨）
# /dev/sdb を実際のディスク名に変更してください
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart primary xfs 0% 100%
sudo mkfs.xfs -f -i size=512 /dev/sdb1
sudo mkdir -p /data/glusterfs/bricks

echo '/dev/sdb1 /data/glusterfs/bricks xfs defaults,inode64,noatime 0 2' | \
  sudo tee -a /etc/fstab
sudo mount -a

sudo mkdir -p /data/glusterfs/bricks/{appdata,asterisk,portainer}


```

### 2-3. Trusted Storage Pool の構成（node01 のみ）

```bash
sudo gluster peer probe node02
sudo gluster peer probe node03
sudo gluster peer probe node04
sudo gluster peer status


```

### 2-4. Replica 3 ボリュームの作成（node01 のみ）

```bash
# アプリケーション汎用データ用ボリューム
sudo gluster volume create gfs-appdata replica 3 \
  node01:/data/glusterfs/bricks/appdata \
  node02:/data/glusterfs/bricks/appdata \
  node03:/data/glusterfs/bricks/appdata force
sudo gluster volume start gfs-appdata

# Asterisk専用ボリューム
sudo gluster volume create gfs-asterisk replica 3 \
  node01:/data/glusterfs/bricks/asterisk \
  node02:/data/glusterfs/bricks/asterisk \
  node03:/data/glusterfs/bricks/asterisk force
sudo gluster volume start gfs-asterisk

# Portainer専用ボリューム
sudo gluster volume create gfs-portainer replica 3 \
  node01:/data/glusterfs/bricks/portainer \
  node02:/data/glusterfs/bricks/portainer \
  node03:/data/glusterfs/bricks/portainer force
sudo gluster volume start gfs-portainer


```

### 2-5. GlusterFS パフォーマンスチューニング（node01）

```bash
for VOL in gfs-appdata gfs-asterisk gfs-portainer; do
  sudo gluster volume set $VOL performance.cache-size 256MB
  sudo gluster volume set $VOL performance.write-behind-window-size 64MB
  sudo gluster volume set $VOL performance.read-ahead on
  sudo gluster volume set $VOL performance.parallel-readdir on
  sudo gluster volume set $VOL cluster.data-self-heal-algorithm full
  sudo gluster volume set $VOL network.ping-timeout 20
done


```

### 2-6. GlusterFS ボリュームのマウント（全ノード）

```bash
sudo mkdir -p /mnt/gluster/{appdata,asterisk,portainer}

sudo tee -a /etc/fstab <<'EOF'

# GlusterFS ボリューム
localhost:/gfs-appdata   /mnt/gluster/appdata   glusterfs defaults,_netdev,backupvolfile-server=node02 0 0
localhost:/gfs-asterisk  /mnt/gluster/asterisk  glusterfs defaults,_netdev,backupvolfile-server=node02 0 0
localhost:/gfs-portainer /mnt/gluster/portainer glusterfs defaults,_netdev,backupvolfile-server=node02 0 0
EOF

sudo mount -a


```

---

## フェーズ3: Docker Swarm クラスター構築

### 3-1. Swarm の初期化（node01 のみ）

```bash
sudo docker swarm init \
  --advertise-addr 192.168.1.241 \
  --listen-addr 192.168.1.241:2377 \
  --default-addr-pool 10.10.0.0/16 \
  --default-addr-pool-mask-length 24


```

### 3-2. マネージャー参加トークンの取得（node01）

```bash
sudo docker swarm join-token manager


```

### 3-3. node02・node03・node04 をマネージャーとして参加（各ノード）

```bash
# 例: node02 で実行
sudo docker swarm join \
  --token SWMTKN-1-[取得したトークン] \
  --advertise-addr 192.168.1.242 \
  --listen-addr 192.168.1.242:2377 \
  192.168.1.241:2377
# ※ node03, node04 も同様にそれぞれのIPで実行


```

### 3-4. ノードラベルの設定（node01）

```bash
sudo docker node update --label-add zone=a node01
sudo docker node update --label-add zone=b node02
sudo docker node update --label-add zone=c node03
sudo docker node update --label-add zone=d node04


```

---

## フェーズ4: ネットワーク設定 (macvlan + VIP + overlay)

### ネットワーク戦略の全体像

```text
┌─────────────────────────────────────────────────────────────────┐
│  Keepalived (VIP)                                               │
│  IP: 192.168.1.240                                              │
│  用途: ホストの物理NICに付与。Swarm Ingress経由で全サービスへ分散 │
│                                                                 │
│  macvlan (swarm-macvlan)                                        │
│  Parent: eth0 / enp1s0 等   Subnet: 192.168.1.0/24              │
│  IP range: 192.168.1.200/32 (192.168.1.200 のみ)                │
│  用途: Asterisk 専用 (192.168.1.200) ← 実LAN IP必須             │
│                                                                 │
│  overlay (swarm-overlay)                                        │
│  Subnet: 10.10.0.0/16   Gateway: 10.10.0.1                      │
│  用途: サービス間通信 (Portainer, DB, Web等)                    │
└─────────────────────────────────────────────────────────────────┘


```

### 4-1. Keepalived による VIP 構成（全ノード）

各ノードで `keepalived.conf` を作成します。NICインターフェース名（`eth0`等）は環境に合わせてください。

**`/etc/keepalived/keepalived.conf` (node01 の例)**

```ini
vrrp_instance VI_1 {
    state MASTER          # node01は MASTER、他は BACKUP
    interface eth0        # 物理NIC名
    virtual_router_id 51
    priority 100          # node02は90, node03は80, node04は70 等と下げる
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass SecretPass123
    }
    virtual_ipaddress {
        192.168.1.240/24  # 共有VIP
    }
}


```

```bash
# 全ノードで起動・有効化
sudo systemctl enable --now keepalived
# node01 でVIPが付与されているか確認
ip addr show


```

### 4-2. macvlan ネットワークの作成

> ⚠️ Asterisk専用の `192.168.1.200` 1つのみを割り当てる設定です。
> ⚠️ **重要**: Swarmモードのmacvlanは、ノードごとの物理NIC名の違い（eth0やenp1s0など）を吸収するため、**「①全ノードでローカル設定を作成」→「②マネージャーでSwarm用ネットワークを作成」の2段階**で行う必要があります。一括で作成しようとすると空の設定が作られ、コンテナが通信できなくなります。

**① 【全ノードで実行】 ローカル設定用ネットワーク（macvlan-config）の作成**

```bash
# 物理NIC名を自動取得
IFACE=$(ip route | grep default | awk '{print $5}')

# ノードごとのローカル設定を作成（--config-only）
sudo docker network create \
  --config-only \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.200/32 \
  --opt parent=$IFACE \
  --opt macvlan_mode=bridge \
  --aux-address="node01=192.168.1.241" \
  --aux-address="node02=192.168.1.242" \
  --aux-address="node03=192.168.1.243" \
  --aux-address="node04=192.168.1.244" \
  macvlan-config

```

*(※必ず node01, node02, node03, node04 の全てでこのコマンドを実行してください)*

**② 【node01 (マネージャー) のみで実行】 Swarm ネットワーク（swarm-macvlan）の作成**

```bash
# ①の設定を参照して、全体で共有する macvlan ネットワークを作成
sudo docker network create \
  --driver macvlan \
  --scope swarm \
  --config-from macvlan-config \
  swarm-macvlan

```

### 4-3. ホスト ↔ macvlan コンテナ間通信の有効化（全ノード）

> Asterisk（192.168.1.200）のみをホストから到達可能にします。

```bash
IFACE=$(ip route | grep default | awk '{print $5}')

# ホスト用macvlanインターフェースを作成
sudo ip link add macvlan-host link $IFACE type macvlan mode bridge
sudo ip addr add 192.168.1.200/32 dev macvlan-host
sudo ip link set macvlan-host up
sudo ip route add 192.168.1.200/32 dev macvlan-host

# 再起動後も永続化
sudo tee /etc/networkd-dispatcher/routable.d/50-macvlan-host.sh <<'EOF'
#!/bin/bash
IFACE=$(ip route | grep default | awk '{print $5}')
ip link add macvlan-host link $IFACE type macvlan mode bridge 2>/dev/null || true
ip addr add 192.168.1.200/32 dev macvlan-host 2>/dev/null || true
ip link set macvlan-host up 2>/dev/null || true
ip route add 192.168.1.200/32 dev macvlan-host 2>/dev/null || true
EOF
sudo chmod +x /etc/networkd-dispatcher/routable.d/50-macvlan-host.sh


```

### 4-4. Overlay ネットワークの作成（node01 のみ）

```bash
sudo docker network create \
  --driver overlay \
  --attachable \
  --subnet=10.10.0.0/16 \
  --gateway=10.10.0.1 \
  --ip-range=10.10.1.0/24 \
  --opt encrypted=true \
  swarm-overlay


```

---

## フェーズ5: Portainer デプロイ (VIP経由)

VIP `192.168.1.240` で待ち受け、Swarm Ingressを使ってPodにルーティングします。

```bash
sudo mkdir -p /opt/stacks/portainer
sudo mkdir -p /mnt/gluster/portainer/data


```

`/opt/stacks/portainer/portainer-stack.yml` を作成：

```yaml
version: "3.9"

services:
  agent:
    image: portainer/agent:lts
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - portainer-agent-network
    deploy:
      mode: global                      # 全ノードに自動展開
      placement:
        constraints:
          - node.platform.os == linux

  portainer:
    image: portainer/portainer-ce:lts
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    ports:
      - target: 9443
        published: 9443
        protocol: tcp
        mode: ingress                   # Swarm Routing Mesh を使用
      - target: 9000
        published: 9000
        protocol: tcp
        mode: ingress
      - target: 8000
        published: 8000
        protocol: tcp
        mode: ingress
    volumes:
      - portainer_data:/data
    networks:
      - portainer-agent-network
      - swarm-overlay
    deploy:
      mode: replicated
      replicas: 1

networks:
  portainer-agent-network:
    driver: overlay
    attachable: true
  swarm-overlay:
    external: true
    name: swarm-overlay

volumes:
  portainer_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/gluster/portainer/data   # GlusterFS上に保存


```

```bash
sudo docker stack deploy -c /opt/stacks/portainer/portainer-stack.yml portainer


```

アクセス先: `https://192.168.1.240:9443`

---

## フェーズ6: Asterisk デプロイ (macvlan)

Asteriskは `192.168.1.200` を専有します。
Asteriskのイメージビルドとスタックファイルの管理はGitHubリポジトリで行い、Portainerからデプロイします。

### 6-1. GlusterFS用ディレクトリの作成

（必要に応じて、永続化するデータを格納するディレクトリを作成しておきます）

```bash
sudo mkdir -p /mnt/gluster/asterisk/{var/lib/asterisk,var/log/asterisk,var/spool/asterisk,var/log/asterisk/cdr-csv}


```

### 6-2. Portainer経由でのデプロイ

1. Portainerにログインし、対象のEnvironmentを開く。
2. 左メニューの **Stacks** > **Add stack** をクリック。
3. **Build method** で「**Repository**」を選択。
4. 以下の情報を入力：
* **Name**: `asterisk`
* **Repository URL**: `https://github.com/warpflow/asterisk.git` (ご自身のフォークがあればそちらに変更)
* **Repository reference**: `refs/heads/main`
* **Compose path**: `stacks/asterisk/asterisk-stack.yml` (※実際のリポジトリ構成に合わせて変更)


5. 「**Deploy the stack**」をクリック。

> ⚠️ **Swarmデプロイ時（compose.yml）の注意：**
> Swarm環境では `compose.yml` の `networks` 定義内で `ipv4_address: 192.168.1.200` と固定IPを直接指定するとエラーになり無視されます。フェーズ4のネットワーク作成時に `--ip-range=192.168.1.200/32` を設定しているため、`compose.yml` 側では単に `- swarm-macvlan` とリスト形式でネットワークを指定するだけで、自動的に `192.168.1.200` が付与されます。

> ⚠️ Asterisk設定ファイル内の `bindaddr` と `externip` (または `externaddr`) は必ず `192.168.1.200` を指定してください。
> `local_net` には物理ネットワーク(`192.168.1.0/24`)とSwarm Overlayネットワーク(`10.10.0.0/16`)の両方を含める必要があります。

---

## フェーズ7: その他サービスのスタック例 (VIP経由)

本構成では、VIP `192.168.1.240` を複数のサービスで **ポートで分けて共有** します。各サービスは `overlay` ネットワークに属し、Swarm Ingressがリクエストを処理します。

### パターン: 複数サービスが 192.168.1.240 を共有

```yaml
version: "3.9"

services:
  # カスタムアプリA
  myapp-a:
    image: nginx:alpine
    ports:
      - target: 80
        published: 8080      # 192.168.1.240:8080 でアクセス
        mode: ingress
    networks:
      - swarm-overlay
    deploy:
      mode: replicated
      replicas: 2            # Podを複数立てても自動でラウンドロビン

  # カスタムアプリB
  myapp-b:
    image: httpd:alpine
    ports:
      - target: 80
        published: 8081      # 192.168.1.240:8081 でアクセス
        mode: ingress
    networks:
      - swarm-overlay
    deploy:
      mode: replicated
      replicas: 1

networks:
  swarm-overlay:
    external: true
    name: swarm-overlay


```

---

## フェーズ8: メンテナンス & トラブルシューティング

### Keepalived のフェイルオーバーテスト

* VIPを持っているノード（例: node01）をシャットダウンするか、`sudo systemctl stop keepalived` を実行します。
* `ping 192.168.1.240` が1秒程度の瞬断で回復することを確認します。
* 別のノード（例: node02）で `ip addr show` を実行し、VIPが移動していることを確認します。

その他のSwarm、GlusterFSのメンテナンス方法は元のガイドと同様です。

---

## 再起動後のメンテナンス

### portainer_agent のエラーで primary に入れない場合
```bash
sudo docker service update --force portainer_agent
```

---

## StacksをNode間で移動させるコマンド

### 移動コマンド
```bash
sudo docker service update --constraint-add node.hostname=={node_name} {service_name}
```

### 戻しコマンド
```bash
sudo docker service update --constraint-rm node.hostname==node03 portainer_portainer
```

---

## デプロイチェックリスト

```text
フェーズ1 — 全ノード共通（node01, node02, node03, node04 で実施）
  ☐ ホスト名の設定
  ☐ /etc/hosts の更新
  ☐ 必要パッケージのインストール（keepalived 追加）
  ☐ カーネルモジュールのロード
  ☐ sysctl 設定
  ☐ UFW ファイアウォール設定（VRRP追加）
  ☐ Docker CE のインストール

フェーズ2 — GlusterFS（node01 から操作）
  ☐ 全ボリュームの起動とマウント確認

フェーズ3 — Docker Swarm
  ☐ 全4ノードがマネージャーとして参加していることを確認

フェーズ4 — ネットワーク (VIP + macvlan + overlay)
  ☐ Keepalived 構成 (VIP: 192.168.1.240 が1つのノードに存在することを確認)
  ☐ swarm-overlay 作成 (node01 のみ)
  ☐ macvlan-config 作成 (全ノードで実行: --config-only) ★重要
  ☐ swarm-macvlan 作成 (node01のみで実行: --config-from) ★重要
  ☐ macvlan-host インターフェース設定 (192.168.1.200へのルート)

フェーズ5, 6, 7 — サービス
  ☐ Portainer スタックのデプロイ (overlay, Ingressポート)
  ☐ Asterisk スタックのデプロイ (compose.ymlでipv4_addressを指定しないこと)
  ☐ その他アプリケーションのデプロイ (overlay, Ingressでポートを分ける)

フェーズ8 — 動作確認
  ☐ ping 192.168.1.200 → Asterisk 応答確認
  ☐ ping 192.168.1.240 → VIP 応答確認
  ☐ https://192.168.1.240:9443 → Portainer アクセス確認


```
