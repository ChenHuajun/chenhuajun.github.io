# 基于Patroni的PostgreSQL高可用环境部署



## 1. 前言

PG的开源HA工具有很多种，下面几种算是比较常用的

- PAF(PostgreSQL Automatic Failover)
- repmgr
- Patroni

它们的比较可以参考: https://scalegrid.io/blog/managing-high-availability-in-postgresql-part-1/



中Patroni采用DCS存储元数据，能够严格的保障元数据的一致性，可靠性高；而且它的功能也比较强大。

因此个人推荐使用Patroni（只有2台机器无法部署etcd的情况可以考虑其它方案）。本文介绍基于Patroni的PostgreSQL高可用的部署。



## 2. 实验环境

**主要软件**

- CentOS 7.8

- PostgreSQL 12

- patroni 1.6.5

- etcd 3.3.25



**机器和vip资源**

- PostgreSQL
  - node1：192.168.234.201 
  - node2：192.168.234.202 
  - node3：192.168.234.203 

- etcd
  - node4：192.168.234.204

- vip
  - 读写VIP：192.168.234.210
  - 只读VIP：192.168.234.211



**环境准备**

所有节点设置时钟同步

```
yum install -y ntpdate
ntpdate time.windows.com && hwclock -w
```

如果使用防火墙需要开放postgres，etcd和patroni的端口。

- postgres:5432
- patroni:8000
- etcd:2379/2380

更简单的做法是将防火墙关闭

```
setenforce 0
sed -i.bak "s/SELINUX=enforcing/SELINUX=permissive/g" /etc/selinux/config
systemctl disable firewalld.service
systemctl stop firewalld.service
iptables -F
```



## 3. etcd部署

因为本文的主题不是etcd的高可用，所以只在node4上部署单节点的etcd用于实验。部署步骤如下



安装需要的包

```
yum install -y gcc python-devel epel-release
```

安装etcd

```
yum install -y etcd
```

编辑etcd配置文件`/etc/etcd/etcd.conf`, 参考配置如下

```
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://192.168.234.204:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://192.168.234.204:2379"
ETCD_NAME="etcd0"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.234.204:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.234.204:2379"
ETCD_INITIAL_CLUSTER="etcd0=http://192.168.234.204:2380"
ETCD_INITIAL_CLUSTER_TOKEN="cluster1"
ETCD_INITIAL_CLUSTER_STATE="new"
```

启动etcd

```
systemctl start etcd
```



设置etcd自启动

```
systemctl enable etcd
```



## 3. PostgreSQL + Patroni HA部署

在需要运行PostgreSQL的实例上安装相关软件



安装PostgreSQL 12

```
yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

yum install -y postgresql12-server postgresql-contrib
```



安装patroni 

```
yum install -y gcc epel-release
yum install -y python-pip python-psycopg2 python-devel

pip install --upgrade pip
pip install --upgrade setuptools
pip install patroni[etcd]
```



创建PostgreSQL数据目录

```
mkdir -p /pgsql/data
chown postgres:postgres -R /pgsql
chmod -R 700 /pgsql/data
```



创建partoni service配置文件`/etc/systemd/system/patroni.service`

```
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target
 
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no
 
[Install]
WantedBy=multi-user.targ
```



创建patroni配置文件`/etc/patroni.yml`,以下是node1的配置示例

```
scope: pgsql
namespace: /service/
name: pg1

restapi:
  listen: 0.0.0.0:8000
  connect_address: 192.168.234.201:8000

etcd:
  host: 192.168.234.204:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        listen_addresses: "0.0.0.0"
        port: 5432
        wal_level: logical
        hot_standby: "on"
        wal_keep_segments: 1000
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"

  initdb:
  - encoding: UTF8
  - locale: C
  - data-checksums

  pg_hba:
  - host replication repl 0.0.0.0/0 md5
  - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.234.201:5432
  data_dir: /pgsql/data
  bin_dir: /usr/pgsql-12/bin

  authentication:
    replication:
      username: repl
      password: "123456"
    superuser:
      username: postgres
      password: "123456"

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

其他PG节点的patroni.yml需要相应修改下面3个参数

- name
- node1~node4分别设置pg1~pg4
- restapi->connect_address
  - 根据各自节点IP设置
- postgresql->connect_address
  - 根据各自节点IP设置



启动patroni

先在node1上启动patroni。

```
systemctl start patroni
```

初次启动patroni时，patroni会初始创建PostgreSQL实例和用户。

```
[root@node1 ~]# systemctl status patroni
● patroni.service - Runners to orchestrate a high-availability PostgreSQL
   Loaded: loaded (/etc/systemd/system/patroni.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2020-09-05 14:41:03 CST; 38min ago
 Main PID: 1673 (patroni)
   CGroup: /system.slice/patroni.service
           ├─1673 /usr/bin/python2 /usr/bin/patroni /etc/patroni.yml
           ├─1717 /usr/pgsql-12/bin/postgres -D /pgsql/data --config-file=/pgsql/data/postgresql.conf --listen_addresses=0.0.0.0 --max_worker_processe...
           ├─1719 postgres: pgsql: logger
           ├─1724 postgres: pgsql: checkpointer
           ├─1725 postgres: pgsql: background writer
           ├─1726 postgres: pgsql: walwriter
           ├─1727 postgres: pgsql: autovacuum launcher
           ├─1728 postgres: pgsql: stats collector
           ├─1729 postgres: pgsql: logical replication launcher
           └─1732 postgres: pgsql: postgres postgres 127.0.0.1(37154) idle
```

再在node2上启动patroni。node2将作为replica加入集群，自动从leader拷贝数据并建立复制。

```
[root@node2 ~]# systemctl status patroni
● patroni.service - Runners to orchestrate a high-availability PostgreSQL
   Loaded: loaded (/etc/systemd/system/patroni.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2020-09-05 16:09:06 CST; 3min 41s ago
 Main PID: 1882 (patroni)
   CGroup: /system.slice/patroni.service
           ├─1882 /usr/bin/python2 /usr/bin/patroni /etc/patroni.yml
           ├─1898 /usr/pgsql-12/bin/postgres -D /pgsql/data --config-file=/pgsql/data/postgresql.conf --listen_addresses=0.0.0.0 --max_worker_processe...
           ├─1900 postgres: pgsql: logger
           ├─1901 postgres: pgsql: startup   recovering 000000010000000000000003
           ├─1902 postgres: pgsql: checkpointer
           ├─1903 postgres: pgsql: background writer
           ├─1904 postgres: pgsql: stats collector
           ├─1912 postgres: pgsql: postgres postgres 127.0.0.1(35924) idle
           └─1916 postgres: pgsql: walreceiver   streaming 0/3000060
```



查看集群状态

```
[root@node2 ~]# patronictl -c /etc/patroni.yml list
+ Cluster: pgsql (6868912301204081018) -------+----+-----------+
| Member |       Host      |  Role  |  State  | TL | Lag in MB |
+--------+-----------------+--------+---------+----+-----------+
|  pg1   | 192.168.234.201 | Leader | running |  1 |           |
|  pg2   | 192.168.234.202 |        | running |  1 |       0.0 |
+--------+-----------------+--------+---------+----+-----------+
```



为了方便日常操作，添加以下环境变量到`~postgres/.bash_profile`，。

```
export PATRONICTL_CONFIG_FILE=/etc/patroni.yml
export PGDATA=/pgsql/data
export PATH=/usr/pgsql-12/bin:$PATH
```

设置postgres拥有sudoer权限

```
echo 'postgres        ALL=(ALL)       NOPASSWD: ALL'> /etc/sudoers.d/postgres
```



## 4. Patroni的自动化动作

Patroni在特定场景下会执行一些自动化动作，目的是为了保障服务的可用性以及防止脑裂。

| 故障位置 | 场景                               | Patroni的动作                                                |
| -------- | ---------------------------------- | ------------------------------------------------------------ |
| 备库     | 备库PG停止                         | 重启PG                                                       |
| 备库     | 停止备库Patroni                    | 同时停止备库PG                                               |
| 备库     | 强杀备库Patroni（或Patroni crash） | 无操作                                                       |
| 备库     | 备库无法连接etcd                   | 无操作                                                       |
| 主库     | 主库PG停止                         | 重启PG，重启超过`master_start_timeout`设定时间，进行主备切换 |
| 主库     | 停止主库Patroni                    | 同时停止库PG，并触发failover                                 |
| 主库     | 强杀主库Patroni（或Patroni crash） | 触发failover                                                 |
| 主库     | 主库无法连接etcd                   | 将主库降级为备库，并触发failover                             |
| -        | etcd集群故障                       | 将主库降级为备库，此时集群中全部都是备库。                   |
| -        | 同步模式下无可用同步备库           | 自动临时切换主库为异步复制，此期间自动failover暂不生效       |



### 4.1 脑裂防护

当Patroni无法连接到etcd时，有一种可能是出现了网络分区。为了防止分区下产生脑裂，如果本机的PG是主库Patroni会把PG降级为备库。

但是，这种做法可能导致在etcd集群故障(包括到etcd的网络故障)时集群中将全部都是备库，业务hang住。为防止出现这种情况可设置比较大的`retry_timeout`（`retry_timeout`控制操作DCS和PostgreSQL的重试时间），确保在触发超时前能解决问题。但这么做也有弊端，增加了脑裂的风险，因为`retry_timeout`的大小某种程度上决定了网络分区时可能出现”双主“的持续时间。



**那么如何更安全地防止脑裂呢?**

有一个很简单的办法，我们设置比较大的`retry_timeout`的同时，把PostgreSQL集群配置成同步模式，代价是降低一点性能。具体设置如下。



**一主一备**

```
retry_timeout:3600
synchronous_mode:true
```

此配置下，当出现网络故障导致主库无法连接etcd，但备库到etcd的访问正常时，备库会被提升为新主。也就是在`retry_timeout`的超时时间到达前出现了"双主"。但由于旧主的PG运行在同步模式下，应用的写入都会被阻塞。新主的PG则被临时切换到了异步模式，应用可以正常写入数据。Patroni通过动态调整`synchronous_standby_names`控制同步异步复制的切换。



我们可以在主节点上阻断etcd和Patroni restapi的端口模拟这种网络故障

```
iptables -I INPUT -p TCP --sport 2379 -j REJECT
iptables -I INPUT -p TCP --dport 2379 -j REJECT
iptables -I INPUT -p TCP --sport 8000 -j REJECT
iptables -I INPUT -p TCP --dport 8000 -j REJECT
```



当etcd本身故障导致主备都无法访问时，在`retry_timeout`的超时时间到达前，主备库都在尝试重连etcd，不会对PG配置进行变更。



如果由于同步备库临时可不用导致Patroni临时把降级成了异步复制，此期间如果主库再发生网络故障，由于集群里没有同步备库，备库不会被提升新主。Patroni会把同步的配置记录到etcd中

正常的同步模式的元数据如下：

```
[root@node4 ~]# etcdctl get /service/cn/sync
{"leader":"pg1","sync_standby":"pg2"}
```

备库无法连接主库后，主库临近降级到异步的元数据如下：

```
[root@node4 ~]# etcdctl get /service/cn/sync
{"leader":"pg1","sync_standby":null}
```



**一主两备**

```
retry_timeout:3600
synchronous_mode:true
synchronous_mode_strict:true
```

一主两备架构下要想达到相同效果，需要额外设置`synchronous_mode_strict=true`，禁止Patroni对同步模式进行降级。这也是最安全的防止脑裂的方式。



**超过3节点的集群**

如果集群中超过3个节点，选择3个节点按`一主两备`的方式，或者选择2个节点按`一主一备`的方式配置。其余节点作为"外挂的从库"，配置为不参与选主，也不作为同步备库。

```
tags:
    nofailover: true
    noloadbalance: false
    clonefrom: false
    nosync: true
```



## 5. 日常操作

日常维护时可以通过`patronictl`命令控制Patroni和PostgreSQL，比如修改PotgreSQL参数。

```
[postgres@node2 ~]$ patronictl --help
Usage: patronictl [OPTIONS] COMMAND [ARGS]...

Options:
  -c, --config-file TEXT  Configuration file
  -d, --dcs TEXT          Use this DCS
  -k, --insecure          Allow connections to SSL sites without certs
  --help                  Show this message and exit.

Commands:
  configure    Create configuration file
  dsn          Generate a dsn for the provided member, defaults to a dsn of...
  edit-config  Edit cluster configuration
  failover     Failover to a replica
  flush        Discard scheduled events (restarts only currently)
  history      Show the history of failovers/switchovers
  list         List the Patroni members for a given Patroni
  pause        Disable auto failover
  query        Query a Patroni PostgreSQL member
  reinit       Reinitialize cluster member
  reload       Reload cluster member configuration
  remove       Remove cluster from DCS
  restart      Restart cluster member
  resume       Resume auto failover
  scaffold     Create a structure for the cluster in DCS
  show-config  Show cluster configuration
  switchover   Switchover to a replica
  version      Output version of patronictl command or a running Patroni...
```



### 5.1 PostgreSQL参数修改

临时修改个别节点的参数，可以使用`ALTER SYSTEM SET ...`执行，比如打开debug日志。对于需要统一配置的参数应该通过`patronictl edit-config`设置，比如修改最大连接数。

```
 patronictl edit-config -s 'postgresql.parameters.max_connections=300'
```

修改最大连接数后需要重启才能生效，因此patroni会设置一个`Pending restart`标志。

```
[postgres@node2 ~]$ patronictl list
+ Cluster: pgsql (6868912301204081018) -------+----+-----------+-----------------+
| Member |       Host      |  Role  |  State  | TL | Lag in MB | Pending restart |
+--------+-----------------+--------+---------+----+-----------+-----------------+
|  pg1   | 192.168.234.201 | Leader | running | 25 |           |        *        |
|  pg2   | 192.168.234.202 |        | running | 25 |       0.0 |        *        |
+--------+-----------------+--------+---------+----+-----------+-----------------+
```

重启集群中所有PG实例

```
 patronictl restart pgsql
```





## 6. 客户端访问配置

HA集群的主节点是动态的，主备发生切换时，客户端对数据库的访问也需要能够动态连接到新主上。有下面几种常见的实现方式

- 多主机URL

- vip
- haproxy



### 6.1 多主机URL

目前pgjdbc和libpq驱动可以在连接字符串中配置多个IP，由驱动识别数据库的主备角色，连接合适的节点。



**JDBC**

JDBC的多主机URL功能全面，支持failover，读写分离和复制均衡。可以通过参数配置不同的连接策略。

- jdbc:postgresql://192.168.234.201:5432,192.168.234.202:5432,192.168.234.203:5432/postgres?targetServerType=primary

  连接主节点(实际是可写的节点)。当出现"双主"甚至"多主"连接第一个发现的可用的主节点

- jdbc:postgresql://192.168.234.201:5432,192.168.234.202:5432,192.168.234.203:5432?targetServerType=preferSecondary&loadBalanceHosts=true

  优先连接备节点，无可用备节点时连接主节点，有多个可用备节点时随机连接其中一个。

- jdbc:postgresql://192.168.234.201:5432,192.168.234.202:5432,192.168.234.203:5432/postgres?targetServerType=any&loadBalanceHosts=true

  随机连接任意一个可用的节点



**libpq**

libpq的多主机URL功能相对pgjdbc弱一点，只支持failover。

- postgres://192.168.234.201:5432,192.168.234.202:5432,192.168.234.203:5432/postgres?target_session_attrs=read-write

  连接主节点(实际是可写的节点)

- postgres://192.168.234.201:5432,192.168.234.202:5432,192.168.234.203:5432/postgres?target_session_attrs=any

  连接任一可用节点



基于libpq实现的其他语言的驱动也可以支持多主机URL，比如python和php。下面是python使用多主机url创建连接的例子

```
import psycopg2

conn=psycopg2.connect("postgres://192.168.234.201:5432,192.168.234.202:5432/postgres?target_session_attrs=read-write&password=123456")
```



### 6.2 VIP(通过callback脚本实现vip漂移）

多主机URL的方式部署简单，但是不是每种语言的驱动都支持，而且如果数据库出现意外的“双主”(HA组件没防护好)，配置多主机URL的客户端在多个主上同时写入的概率比较高。而如果客户端通过VIP的方式访问则在VIP上又多了一层防护。

Patroni支持用户配置在特定事件发生时触发的回调脚本。因此我们可以配置一个回调，在主备切换后动态加载vip。



准备加载vip的回调脚本`/pgsql/loadvip.sh`

```
#!/bin/bash

VIP=192.168.234.210
GATEWAY=192.168.234.2
DEV=ens33

action=$1
role=$2
cluster=$3

log()
{
  echo "loadvip: $*"|logger
}

load_vip()
{
ip a|grep -w ${DEV}|grep -w ${VIP} >/dev/null
if [ $? -eq 0 ] ;then
  log "vip exists, skip load vip"
else
  sudo ip addr add ${VIP}/32 dev ${DEV} >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to add vip ${VIP} at dev ${DEV} rc=$rc"
    exit 1
  fi

  log "added vip ${VIP} at dev ${DEV}"

  arping -U -I ${DEV} -s ${VIP} ${GATEWAY} -c 5 >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to call arping to gateway ${GATEWAY} rc=$rc"
    exit 1
  fi
  
  log "called arping to gateway ${GATEWAY}"
fi
}

unload_vip()
{
ip a|grep -w ${DEV}|grep -w ${VIP} >/dev/null
if [ $? -eq 0 ] ;then
  sudo ip addr del ${VIP}/32 dev ${DEV} >/dev/null
  rc=$?
  if [ $rc -ne 0 ] ;then
    log "fail to delete vip ${VIP} at dev ${DEV} rc=$rc"
    exit 1
  fi

  log "deleted vip ${VIP} at dev ${DEV}"
else
  log "vip not exists, skip delete vip"
fi
}

log "loadvip start args:'$*'"

case $action in
  on_start|on_restart|on_role_change)
    case $role in
      master)
        load_vip
        ;;
      replica)
        unload_vip
        ;;
      *)
        log "wrong role '$role'"
        exit 1
        ;;
    esac
    ;;
  *)
    log "wrong action '$action'"
    exit 1
    ;;
esac
```



修改patroni配置文件`/etc/patroni.yml`，配置回调函数

```
postgresql:
...
  callbacks:
    on_start: /bin/bash /pgsql/loadvip.sh
    on_restart: /bin/bash /pgsql/loadvip.sh
    on_role_change: /bin/bash /pgsql/loadvip.sh
```

所有节点的patroni配置文件都修改后，重新加载patroni配置

```
patronictl reload pgsql
```

执行switchover后，可以看到vip发生可漂移

/var/log/messages:

```
Sep  5 21:32:24 localvm postgres: loadvip: loadvip start args:'on_role_change master pgsql'
Sep  5 21:32:24 localvm systemd: Started Session c7 of user root.
Sep  5 21:32:24 localvm postgres: loadvip: added vip 192.168.234.210 at dev ens33
Sep  5 21:32:25 localvm patroni: 2020-09-05 21:32:25,415 INFO: Lock owner: pg1; I am pg1
Sep  5 21:32:25 localvm patroni: 2020-09-05 21:32:25,431 INFO: no action.  i am the leader with the lock
Sep  5 21:32:28 localvm postgres: loadvip: called arping to gateway 192.168.234.2
```



**注意**：

如果直接停止主库上的patroni，vip不会被摘掉。主库上的patroni被停掉后会触发备库failover成为新主，此时新旧主2台机器上都又vip，但是由于新主执行了arping，一般不会影响应用访问。



### 6.3 VIP(通过keepalived实现VIP漂移）

Patroni提供了用于健康检查的REST API，可以根据节点角色返回正常(**200**)和异常的HTTP状态码

- `GET /` 或 `GET /leader`

  运行中且是leader节点

- `GET /replica`

  运行中且是replica角色，且没有设置tag noloadbalance

- `GET /read-only`

  和`GET /replica`类似，但是包含leader节点

使用REST API，Patroni可以和外部组件搭配使用。比如可以配置keepalived动态绑vip。

下面的例子在一主一备集群(node1和node2)中动态在备节点上绑只读vip（192.168.234.211），当备节点故障时则将只读vip绑在主节点上。



安装keepalived

```
yum install -y keepalived
```



准备keepalived配置文件`/etc/keepalived/keepalived.conf`

```
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_leader {
    script "/usr/bin/curl -s http://127.0.0.1:8000/leader -v 2>&1|grep '200 OK' >/dev/null"
    interval 2
    weight 10
}
vrrp_script check_replica {
    script "/usr/bin/curl -s http://127.0.0.1:8000/replica -v 2>&1|grep '200 OK' >/dev/null"
    interval 2
    weight 5
}
vrrp_script check_can_read {
    script "/usr/bin/curl -s http://127.0.0.1:8000/read-only -v 2>&1|grep '200 OK' >/dev/null"
    interval 2
    weight 10
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 211
    priority 100
    advert_int 1
    track_script {
        check_can_read
        check_replica
    }
    virtual_ipaddress {
       192.168.234.211
    }
}
```

启动keepalived

```
systemctl start keepalived
```



上面的配置方法也可以用于读写vip的漂移，只要把`track_script`中的脚本换成`check_leader`即可。但是在网络抖动或其它临时故障时keepalived管理的vip容易飘，因此建议尽量不要直接使用keepalived管理数据库的读写vip。如果有多个备库，也可以在keepalived中配置LVS对所有备库进行负载均衡，过程就不展开了。



### 6.4 haproxy

haproxy的配置方案

安装haproxy

```
yum install -y haproxy
```



编辑haproxy配置文件`/etc/haproxy/haproxy.cfg`

```
global
    maxconn 100
    log     127.0.0.1 local2

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen pgsql
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgresql_192.168.234.201_5432 192.168.234.201:5432 maxconn 100 check port 8000
    server postgresql_192.168.234.202_5432 192.168.234.202:5432 maxconn 100 check port 8000
    server postgresql_192.168.234.203_5432 192.168.234.203:5432 maxconn 100 check port 8000

listen pgsql_read
    bind *:6000
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgresql_192.168.234.201_5432 192.168.234.201:5432 maxconn 100 check port 8000
    server postgresql_192.168.234.202_5432 192.168.234.202:5432 maxconn 100 check port 8000
    server postgresql_192.168.234.203_5432 192.168.234.203:5432 maxconn 100 check port 8000
```

如果只有2个节点，上面的`GET /replica `需要改成`GET /read-only`，否则备库故障时就无法提供只读访问了，但是这样配置主库也会参与读，不能完全分离主库的读负载。



haproxy自身也需要高可用，可以把haproxy部署在node1和node2 2台机器上，通过keepalived控制vip在node1和node2上漂移。

准备keepalived配置文件`/etc/keepalived/keepalived.conf`

```
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_haproxy {
    script "pgrep -x haproxy"
    interval 2
    weight 10
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 210
    priority 100
    advert_int 1
    track_script {
        check_haproxy
    }
    virtual_ipaddress {
       192.168.234.210
    }
}
```



启动haproxy

```
systemctl start haproxy
```



启动keepalived

```
systemctl start keepalived
```



做个简单的测试，从node4上通过haproxy的5000端口分别访问postgres，会连到主库上

```
[postgres@node4 ~]$ psql "host=192.168.234.210 port=5000 password=123456" -c 'select inet_server_addr()'
 inet_server_addr
------------------
 192.168.234.201
(1 row)
```

通过haproxy的6000端口访问postgres，会轮询连接2个备库

```
[postgres@node4 ~]$ psql "host=192.168.234.210 port=6000 password=123456" -c 'select inet_server_addr()'
 inet_server_addr
------------------
 192.168.234.202
(1 row)

[postgres@node4 ~]$ psql "host=192.168.234.210 port=6000 password=123456" -c 'select inet_server_addr()'
 inet_server_addr
------------------
 192.168.234.203
(1 row)
```



haproxy部署后，可以通过它的web接口 http://192.168.234.201:7000/查看统计数据



### 7. 参考

- https://patroni.readthedocs.io/en/latest/
- http://blogs.sungeek.net/unixwiz/2018/09/02/centos-7-postgresql-10-patroni/
- https://scalegrid.io/blog/managing-high-availability-in-postgresql-part-1/
- https://jdbc.postgresql.org/documentation/head/connect.html#connection-parameters

- https://www.percona.com/blog/2019/10/23/seamless-application-failover-using-libpq-features-in-postgresql/

