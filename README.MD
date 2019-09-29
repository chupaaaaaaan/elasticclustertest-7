# Elasticsearch7.3クラスタ構成（マルチDockerホスト） 考慮点

## マシン
AWS EC2インスタンス3台（r5a.largeインスタンス）

| IP           | etcdクラスタ メンバ名 |
|:-------------|:----------------------|
| 192.168.0.7  | `etcd01`              |
| 192.168.0.24 | `etcd02`              |
| 192.168.0.28 | `etcd03`              |

##  etcd インストール
sudo yum install -y etcd &&
sudo sh -c "[ -f /etc/etcd/etcd.conf.orig ] || mv /etc/etcd/etcd.conf{,.orig}"

### No.1
sudo sh -c "cat <<EOF > /etc/etcd/etcd.conf 
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_NAME="etcd01"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.7:2380"
ETCD_LISTEN_PEER_URLS="http://192.168.0.7:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.7:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.7:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd01=http://192.168.0.7:2380,etcd02=http://192.168.0.24:2380,etcd03=http://192.168.0.28:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF"

### No.2
sudo sh -c "cat <<EOF > /etc/etcd/etcd.conf 
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_NAME="etcd02"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.24:2380"
ETCD_LISTEN_PEER_URLS="http://192.168.0.24:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.24:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.24:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd01=http://192.168.0.7:2380,etcd02=http://192.168.0.24:2380,etcd03=http://192.168.0.28:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF"

### No.3
sudo sh -c "cat <<EOF > /etc/etcd/etcd.conf 
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_NAME="etcd03"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.28:2380"
ETCD_LISTEN_PEER_URLS="http://192.168.0.28:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.28:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.28:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="etcd01=http://192.168.0.7:2380,etcd02=http://192.168.0.24:2380,etcd03=http://192.168.0.28:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF"

### End every host settings

SERVICE_FILE=/usr/lib/systemd/system/etcd.service &&
sudo sh -c "[ -f ${SERVICE_FILE}.orig ] || cp -p ${SERVICE_FILE}{,.orig}" &&
sudo sed -i -e 's@^ExecStart.\+$@ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd"@' ${SERVICE_FILE} &&
sudo systemctl enable etcd &&
sudo systemctl start etcd &&
unset SERVICE_FILE


## docker インストール
sudo yum install -y yum-utils device-mapper-persistent-data lvm2 &&
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo &&
sudo yum install -y docker-ce docker-ce-cli containerd.io &&
sudo usermod -aG docker centos

SERVICE_FILE=/usr/lib/systemd/system/docker.service &&
sudo sh -c "[ -f ${SERVICE_FILE}.orig ] || cp -p ${SERVICE_FILE}{,.orig}"

### No.1
sudo sed -i -e 's@^ExecStart.\+$@ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock -H fd:// --cluster-store=etcd://192.168.0.7:2379 --cluster-store=etcd://192.168.0.24:2379 --cluster-store=etcd://192.168.0.28:2379 --cluster-advertise=192.168.0.7:2375 --containerd=/run/containerd/containerd.sock@' ${SERVICE_FILE}

### No.2
sudo sed -i -e 's@^ExecStart.\+$@ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock -H fd:// --cluster-store=etcd://192.168.0.7:2379 --cluster-store=etcd://192.168.0.24:2379 --cluster-store=etcd://192.168.0.28:2379 --cluster-advertise=192.168.0.24:2375 --containerd=/run/containerd/containerd.sock@' ${SERVICE_FILE}

### No.3
sudo sed -i -e 's@^ExecStart.\+$@ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock -H fd:// --cluster-store=etcd://192.168.0.7:2379 --cluster-store=etcd://192.168.0.24:2379 --cluster-store=etcd://192.168.0.28:2379 --cluster-advertise=192.168.0.28:2375 --containerd=/run/containerd/containerd.sock@' ${SERVICE_FILE}

### End every host settings

unset SERVICE_FILE &&
sudo systemctl enable docker &&
sudo systemctl start docker


## docker-compose インストール
curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-Linux-x86_64 > docker-compose &&
chmod 555 docker-compose &&
sudo mv docker-compose /usr/local/bin


## elasticsearch関連設定
### No.1
sudo mkdir -p   /{esdata,eslogs}/{master01,data01-01,data01-02} &&
sudo chmod 777  /{esdata,eslogs}/{master01,data01-01,data01-02} &&
sudo chgrp 1000 /{esdata,eslogs}/{master01,data01-01,data01-02}

### No.2
sudo mkdir -p   /{esdata,eslogs}/{master02,data02-01,data02-02} &&
sudo chmod 777  /{esdata,eslogs}/{master02,data02-01,data02-02} &&
sudo chgrp 1000 /{esdata,eslogs}/{master02,data02-01,data02-02}

### No.3
sudo mkdir -p   /{esdata,eslogs}/master03 &&
sudo chmod 777  /{esdata,eslogs}/master03 &&
sudo chgrp 1000 /{esdata,eslogs}/master03

### End every host settings

sudo -c "echo 'vm.max_map_count=262144' >> /etc/sysctl.conf" &&
sudo sysctl -p


## クリンナップ＆再起動
docker-compose down
rm -rf /{esdata,eslogs}/*/*
docker-compose up -d

## 参考文献

* https://www.elastic.co/guide/en/elasticsearch/reference/7.3/docker.html
* https://knowledge.sakura.ad.jp/4786/
* https://docs.docker.com/network/
* https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-settings.html
* https://qiita.com/smallpalace/items/9f41ac3ddcc95cfddb28
* https://qiita.com/hana_shin/items/602f98bd9b153d22e50c
* https://qiita.com/ttsubo/items/1257fe914744d00b4ed8
* https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/clustering.md
* http://docs.docker.jp/engine/admin/systemd.html
* http://docs.docker.jp/compose/networking.html
* https://www.elastic.co/guide/en/elasticsearch/reference/7.3/_memory_lock_check.html
