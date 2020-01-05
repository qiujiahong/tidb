# 部署

## 规划 

* 建议

| 组件 | CPU  | 内存   | 本地存储     | 网络     | 实例数量(最低要求)  |
|------|------|--------|--------------|--------|---------------------|
| TiDB | 8核+ | 16 GB+ | 无特殊要求   | 千兆网卡 | 1（可与 PD 同机器）   |
| PD   | 4核+ | 8 GB+  | SAS, 200 GB+ | 千兆网卡 | 1（可与 TiDB 同机器） |
| TiKV | 8核+ | 32 GB+ | SSD, 200 GB+ | 千兆网卡 | 3                   |

>  8c 32GB 200GB SSD三台，本文如上使用四台主机部署。

* 实际 


| 节点  | ip              | 部署组件 | 配置            |
|-------|-----------------|----------|-----------------|
| node1 | 10.170.0.11 | PD、TiDB  | 8C 32G 200G SSD |
| node2 | 10.170.0.12 | TiKV     | 8C 32G 200G SSD |
| node3 | 10.140.0.3 | TiKV     | 8C 32G 200G SSD |
| node4 | 10.140.0.4| TiKV     | 8C 32G 200G SSD |

> 其中node1作为中控机，与其他机器配置免密登录



## 中控机配置host 

```bash
sed -i "/node1/d" /etc/hosts
sed -i "/node2/d" /etc/hosts
sed -i "/node3/d" /etc/hosts
sed -i "/node4/d" /etc/hosts
cat << EOF >> /etc/hosts
10.170.0.11 node1  
10.170.0.12 node2  
10.140.0.3 node3  
10.140.0.4 node4  
EOF
```

## 中控机安装辅助软件



```bash 

yum install -y wget vim curl git
wget https://download.pingcap.org/ansible-system-rpms.el7.tar.gz
# 在中控机上安装系统依赖包：
tar -xzvf ansible-system-rpms.el7.tar.gz &&
cd ansible-system-rpms.el7 &&
chmod u+x install_ansible_system_rpms.sh &&
./install_ansible_system_rpms.sh
pip -V

```


## 中控机上创建 tidb 用户，并生成 SSH key

```bash 
# 创建 tidb 用户
useradd -m -d /home/tidb tidb
# 设置 tidb 用户密码。
passwd tidb
# 配置 tidb 用户 sudo 免密码，将 ``tidb ALL=(ALL) NOPASSWD: ALL``添加到文件末尾即可。
visudo

# 生成 SSH key。
su - tidb
ssh-keygen -t rsa


```

## 中控机器上离线安装 Ansible 及其依赖

```bash 
# 下载离线安装包,下载机上下载
wget https://download.pingcap.org/ansible-2.5.0-pip.tar.gz
# 离线安装 Ansible 及相关依赖：
tar -xzvf ansible-2.5.0-pip.tar.gz &&
cd ansible-2.5.0-pip/ &&
chmod u+x install_ansible.sh &&
./install_ansible.sh
# 验证安装情况
ansible --version
```

## 下载机上下载 TiDB Ansible 及 TiDB 安装包

```BASH 
# 下载TiDB Ansible
su tidb
cd ~
tag=v3.0.2
git clone -b $tag https://github.com/pingcap/tidb-ansible.git
# 执行ansible下载安装包
cd tidb-ansible &&
ansible-playbook local_prepare.yml
# 将执行完以上命令之后的 tidb-ansible 文件夹拷贝到中控机 /home/tidb 目录下，文件属主权限需是 tidb 用户。

```

## 在中控机上配置部署机器 SSH 互信及 sudo 规则

```bash 
# 将你的部署目标机器 IP 添加到 hosts.ini 文件的 [servers] 区块下。
cd /home/tidb/tidb-ansible && \
vi hosts.ini
```

```bash 
[servers]
10.170.0.11
10.170.0.12
10.140.0.3
10.140.0.4

[all:vars]
username = tidb
ntp_server = pool.ntp.org
```

```bash 
# 执行以下命令，按提示输入部署目标机器的 root 用户密码。
ansible-playbook -i hosts.ini create_users.yml -u root -k
# 该步骤将在部署目标机器上创建 tidb 用户，并配置 sudo 规则，配置中控机与部署目标机器之间的 SSH 互信。
```


## 编辑 inventory.ini 文件，分配机器资源



```bash 
cat << EOF > inventory.ini
## TiDB Cluster Part
[tidb_servers]
10.170.0.11

[tikv_servers]
10.170.0.12
10.140.0.3
10.140.0.4

[pd_servers]
10.170.0.11

[spark_master]

[spark_slaves]

[lightning_server]

[importer_server]

## Monitoring Part
# prometheus and pushgateway servers
[monitoring_servers]
10.170.0.11

[grafana_servers]
10.170.0.11

# node_exporter and blackbox_exporter servers
[monitored_servers]
10.170.0.11
10.170.0.12
10.140.0.3
10.140.0.4

[alertmanager_servers]
# 10.140.0.4

[kafka_exporter_servers]

## Binlog Part
[pump_servers]

[drainer_servers]

## Group variables
[pd_servers:vars]
# location_labels = ["zone","rack","host"]

## Global variables
[all:vars]
deploy_dir = /home/tidb/deploy

## Connection
# ssh via normal user
ansible_user = tidb

cluster_name = test-cluster

tidb_version = v3.0.2

# process supervision, [systemd, supervise]
process_supervision = systemd

timezone = Asia/Shanghai

enable_firewalld = False
# check NTP service
enable_ntpd = True
set_hostname = False

## binlog trigger
enable_binlog = False

# kafka cluster address for monitoring, example:
# kafka_addrs = "192.168.0.11:9092,192.168.0.12:9092,192.168.0.13:9092"
kafka_addrs = ""

# zookeeper address of kafka cluster for monitoring, example:
# zookeeper_addrs = "192.168.0.11:2181,192.168.0.12:2181,192.168.0.13:2181"
zookeeper_addrs = ""

# enable TLS authentication in the TiDB cluster
enable_tls = False

# KV mode
deploy_without_tidb = False

# wait for region replication complete before start tidb-server.
wait_replication = True

# Optional: Set if you already have a alertmanager server.
# Format: alertmanager_host:alertmanager_port
alertmanager_target = ""

grafana_admin_user = "admin"
grafana_admin_password = "admin"


### Collect diagnosis
collect_log_recent_hours = 2

enable_bandwidth_limit = True
# default: 10Mb/s, unit: Kbit/s
collect_bandwidth_limit = 10000
EOF

# 执行以下命令，如果所有 server 均返回 tidb，表示 SSH 互信配置成功：
ansible -i inventory.ini all -m shell -a 'whoami'
# 执行以下命令，如果所有 server 均返回 root，表示 tidb 用户 sudo 免密码配置成功。
ansible -i inventory.ini all -m shell -a 'whoami' -b

```


## 安装文件  

```bash 
# 执行 local_prepare.yml playbook，联网下载 TiDB binary 到中控机。 如果离线安装则在联网电脑上执行之后再上传上去
# ansible-playbook local_prepare.yml
# 初始化系统环境，修改内核参数。
# ansible-playbook bootstrap.yml
# 如下是测试模式，将不会检查内存大小
ansible-playbook bootstrap.yml --extra-vars "dev_mode=True"
# 部署 TiDB 集群软件。
ansible-playbook deploy.yml
# 启动 TiDB 集群。
ansible-playbook start.yml

```

## 测试集群

TiDB 兼容 MySQL，因此可使用 MySQL 客户端直接连接 TiDB。推荐配置负载均衡以提供统一的 SQL 接口。

1、使用 MySQL 客户端连接 TiDB 集群。TiDB 服务的默认端口为 4000。
```BASH 
mysql -u root -h 172.16.10.1 -P 4000
```
2、通过浏览器访问监控平台。
> 地址：http://172.16.10.1:3000
> 默认帐号与密码：admin；admin