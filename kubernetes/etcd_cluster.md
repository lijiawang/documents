# Etcd Cluster with SSL

## 一、环境

### 1. 节点

| name      | ip            |
|:---------:|:---------:    |
| etcd1     | 192.168.20.31 |
| etcd2     | 192.168.20.32 |
| etcd3     | 192.168.20.33 |

### 2. 版本

- etcd 3.2.5+

### 3. Firewalld
- ##### Stop & Disable
- ##### 由 k8s 自行管理
- #### 如单独部署 Etcd 集群，则不必关闭 firewalld

## 二、安装
在各节点依次执行 yum install -y etcd 进行安装

    yum install -y etcd

## 三、配置
### 1.  修改配置文件 /etc/etcd/etcd.conf

    [member]
    ETCD_NAME=etcd1
    ETCD_DATA_DIR="/var/lib/etcd/etcd"
    ETCD_LISTEN_PEER_URLS="https://192.168.20.31:2380"
    ETCD_LISTEN_CLIENT_URLS="https://192.168.20.31:2379"
    [cluster]
    ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.20.31:2380"
    ETCD_INITIAL_CLUSTER="etcd1=https://192.168.20.31:2380"
    ETCD_INITIAL_CLUSTER_STATE="new"
    ETCD_ADVERTISE_CLIENT_URLS="https://192.168.20.31:2379"
    [security]
    ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
    ETCD_KEY_FILE="/etc/etcd/ssl/etcd.key"
    ETCD_CLIENT_CERT_AUTH="true"
    ETCD_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
    ETCD_AUTO_TLS="true"
    ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
    ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd.key"
    ETCD_PEER_CLIENT_CERT_AUTH="true"
    ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
    ETCD_PEER_AUTO_TLS="true"

  - 证书路径请根据证书实际目录对照修改
  - ##### ETCD_INITIAL_CLUSTER_STATE 设置为 new
  - ##### ETCD_DATA_DIR 指定数据存放路径，在生产环境集群推荐使用高性能SSD

### 3. 修改文件 /usr/lib/systemd/system/etcd.service
- ##### File: /usr/lib/systemd/system/etcd.service

      [Unit]
      Description=Etcd Server
      After=network.target
      After=network-online.target
      Wants=network-online.target

      [Service]
      Type=notify
      WorkingDirectory=/var/lib/etcd/
      EnvironmentFile=-/etc/etcd/etcd.conf
      User=etcd

      ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd \
          --name=\"${ETCD_NAME}\" \
          --cert-file=\"${ETCD_CERT_FILE}\" \
          --key-file=\"${ETCD_KEY_FILE}\" \
          --peer-cert-file=\"${ETCD_PEER_CERT_FILE}\" \
          --peer-key-file=\"${ETCD_PEER_KEY_FILE}\" \
          --trusted-ca-file=\"${ETCD_TRUSTED_CA_FILE}\" \
          --peer-trusted-ca-file=\"${ETCD_PEER_TRUSTED_CA_FILE}\" \
          --initial-advertise-peer-urls=\"${ETCD_INITIAL_ADVERTISE_PEER_URLS}\" \
          --listen-peer-urls=\"${ETCD_LISTEN_PEER_URLS}\" \
          --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\" \
          --advertise-client-urls=\"${ETCD_ADVERTISE_CLIENT_URLS}\" \
          --initial-cluster-token=\"${ETCD_INITIAL_CLUSTER_TOKEN}\" \
          --initial-cluster=\"${ETCD_INITIAL_CLUSTER}\" \
          --initial-cluster-state=\"${ETCD_INITIAL_CLUSTER_STATE}\" \
          --data-dir=\"${ETCD_DATA_DIR}\""

      Restart=on-failure
      LimitNOFILE=65536

      [Install]
      WantedBy=multi-user.target

### 4. 依次修改3个节点的 /etc/etcd/etc.conf 和 /usr/lib/systemd/system/etcd.service 文件

### 5. 重新载入配置
- #### 由于修改了 systemd 配置，所以需要重新载入配置

       systemctl daemon-reload

## 四、启动 & 初始化集群
### 1. 启动第一个 etcd ( *etcd1* )
- #### 命令

       systemctl start etcd

- #### 查看 member

       etcdctl --endpoints=https://192.168.50.55:2379 \
                              --ca-file=/etc/kubernetes/ssl/ca.pem \
                              --cert-file=/etc/etcd/ssl/etcd.pem \
                              --key-file=/etc/etcd/ssl/etcd.key \
                              member list

### 2. 将 etcd2 节点(不启动)加入集群
- #### 加入集群

       etcdctl --endpoints=https://192.168.50.55:2379 \
                              --ca-file=/etc/kubernetes/ssl/ca.pem \
                              --cert-file=/etc/etcd/ssl/etcd.pem \
                              --key-file=/etc/etcd/ssl/etcd.key \
                              member add etcd2 https://192.168.50.56:2380

      Added member named etcd2 with ID 669b333wwf2ce34 to cluster

      ETCD_NAME="etcd2"
      ETCD_INITIAL_CLUSTER="etcd2=https://192.168.50.56:2380,etcd1=https://192.168.50.55:2380"
      ETCD_INITIAL_CLUSTER_STATE="existing"

- #### 按上述提示修改 etcd2:/etc/etcd/etcd.conf 配置

- #### 启动 etcd2

      [root@50-56 ~]# systemctl start etcd

- #### 启用 etcd2

      [root@50-56 ~]# systemctl enable etcd

### 3. 将 etcd3 节点(不启动)加入集群

       etcdctl --endpoints=https://192.168.50.55:2379 \
                              --ca-file=/etc/kubernetes/ssl/ca.pem \
                              --cert-file=/etc/etcd/ssl/etcd.pem \
                              --key-file=/etc/etcd/ssl/etcd.key \
                              member add etcd3 https://192.168.50.57:2380

      Added member named etcd3 with ID c2334e2016572cb to cluster

      ETCD_NAME="etcd3"
      ETCD_INITIAL_CLUSTER="etcd3=https://192.168.50.57:2380,etcd2=https://192.168.50.56:2380,etcd1=https://192.168.50.55:2380"
      ETCD_INITIAL_CLUSTER_STATE="existing"

  - #### 按上述提示修改 etcd3:/etc/etcd/etcd.conf 配置

  - #### 启动 etcd3

        [root@50-57 ~]# systemctl start etcd

  - #### 启用 etcd3

        [root@50-57 ~]# systemctl enable etcd

### 4. 将添加 etcd3 时的输出增加到 etcd2 & etcd3 节点 /etc/etcd/etcd.conf 文件中，文件示例如下
- #### File: /etc/etcd/etcd.conf

      [member]
      ETCD_NAME=etcd2
      ETCD_DATA_DIR="/var/lib/etcd/etcd2"
      ETCD_LISTEN_PEER_URLS="https://192.168.50.55:2380"
      ETCD_LISTEN_CLIENT_URLS="https://192.168.50.55:2379"
      [cluster]
      ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.50.55:2380"
      ETCD_INITIAL_CLUSTER="etcd1=https://192.168.50.55:2380,etcd2=https://192.168.50.56:2380,etcd3=https://192.168.50.57:2380"
      ETCD_INITIAL_CLUSTER_STATE="exsting"
      ETCD_ADVERTISE_CLIENT_URLS="https://192.168.50.55:2379"
      [security]
      ETCD_CERT_FILE="/etc/etcd/ssl/etcd.pem"
      ETCD_KEY_FILE="/etc/etcd/ssl/etcd.key"
      ETCD_CLIENT_CERT_AUTH="true"
      ETCD_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
      ETCD_AUTO_TLS="true"
      ETCD_PEER_CERT_FILE="/etc/etcd/ssl/etcd.pem"
      ETCD_PEER_KEY_FILE="/etc/etcd/ssl/etcd.key"
      ETCD_PEER_CLIENT_CERT_AUTH="true"
      ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
      ETCD_PEER_AUTO_TLS="true"

  - ##### 注意修改 ETCD_NAME

### 5. 将 etcd2 & etcd3 节点启动
- #### 启动 & 启用

      systemctl start etcd && systemctl enable etcd

      systemctl start etcd && systemctl enable etcd

### 6. 修改 /etc/etcd/etcd.conf 文件，保持一致
- #### 将三个节点配置文件中以下变量替换为:

      ETCD_INITIAL_CLUSTER="etcd1=https://192.168.50.55:2380,etcd2=https://192.168.50.56:2380,etcd3=https://192.168.50.57:2380"
      ETCD_INITIAL_CLUSTER_STATE="existing"

- #### 然后逐个重启各节点

### 7. 查看集群成员
- #### 命令行需指定证书

      etcdctl --endpoints=https://192.168.20.31:2379 \
              --ca-file=/etc/kubernetes/ssl/ca.pem \
              --cert-file=/etc/etcd/ssl/etcd.pem \
              --key-file=/etc/etcd/ssl/etcd.key \
              member list

## 六、参考资料

1. [Kubernetes with TLS](https://github.com/Statemood/documents/blob/master/kubernetes/)