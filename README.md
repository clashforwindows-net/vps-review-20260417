# VPS 数据库选型与高可用部署实战（2026）

> 在 VPS 上把数据库跑稳，比"装上去能连"难一个量级。本篇面向站长与开发者，系统讲清 MySQL / PostgreSQL / Redis 在单台与多台 VPS 上的选型、调优、主从复制与高可用（HA）部署，附可直接落地的部署脚本、参数模板、监控告警与故障切换方案，让你用最低成本获得接近云厂商 RDS 的可靠性。

本仓库是 clashforwindows-net 系列 VPS 评测的第 17 篇，专题聚焦 **数据库选型与高可用部署**。如果你还在用 SQLite 扛生产流量，或把 MySQL 装在和系统盘同盘的 1 核机器上，这篇能帮你避开大多数生产事故。数据库是服务的"心脏"，心脏停了，前面再华丽的负载均衡也白搭——所以本文把"选型"和"不丢数据"放在同等重要的位置，并给出从单机到主从 HA 的完整演进路径。

## 目录

- [先想清楚：你需要哪种数据库](#先想清楚你需要哪种数据库)
- [三大数据库横评与选型矩阵](#三大数据库横评与选型矩阵)
- [VPS 配置与磁盘基线建议](#vps-配置与磁盘基线建议)
- [MySQL 单机部署与参数调优](#mysql-单机部署与参数调优)
- [PostgreSQL 单机部署与调优](#postgresql-单机部署与调优)
- [Redis 部署与持久化策略](#redis-部署与持久化策略)
- [主从复制：MySQL 实战](#主从复制mysql-实战)
- [PostgreSQL 流复制 standby](#postgresql-流复制-standby)
- [Redis 主从 + 哨兵](#redis-主从--哨兵)
- [高可用方案对比：Keepalived / Orchestrator / Patroni](#高可用方案对比keepalived--orchestrator--patroni)
- [连接池与性能压测](#连接池与性能压测)
- [数据库安全加固](#数据库安全加固)
- [监控指标与告警](#监控指标与告警)
- [慢查询与索引优化实战](#慢查询与索引优化实战)
- [容量规划与扩容](#容量规划与扩容)
- [故障诊断速查](#故障诊断速查)
- [备份与 PITR（按时间点恢复）](#备份与-pitr按时间点恢复)
- [故障切换演练](#故障切换演练)
- [推荐 VPS](#推荐-vps)
- [相关资源](#相关资源)
- [常见问题](#常见问题)
- [免责声明](#免责声明)

## 先想清楚：你需要哪种数据库

很多人在 VPS 上"无脑装 MySQL"，其实数据库选型应取决于数据模型与一致性需求：

- **关系型、强一致、事务**：MySQL / PostgreSQL（订单、用户、财务）。
- **缓存、会话、排行榜、队列**：Redis（高并发读、低延迟）。
- **文档/灵活结构**：MongoDB（内容型、配置型）。
- **全文检索**：Elasticsearch/Meilisearch（搜索场景）。

新手最常犯的错：用 MySQL 存会话（应放 Redis）、用 Redis 存需要持久化的钱（应放关系库）。职责分离是稳定性的第一步。另一个常见误区是"所有数据都进一张大表"，正确做法是按访问频率拆冷热：热数据进 Redis，温数据进关系库加索引，冷数据归档到对象存储或单独的历史表。

## 三大数据库横评与选型矩阵

| 维度 | MySQL 8 | PostgreSQL 16 | Redis 7 |
|------|---------|---------------|---------|
| 定位 | 通用关系库 | 通用关系库（更严谨） | 内存 KV/数据结构 |
| 事务 | 支持（InnoDB） | 支持（更强隔离） | 单命令原子 |
| 复杂查询 | 好 | 极强（CTE/窗口函数） | 弱 |
| 扩展生态 | 极大 | 大（扩展丰富） | 中等 |
| 云托管成熟度 | 高 | 高 | 高 |
| 单机并发读 | 高 | 高 | 极高（内存级） |
| 适合 | 大多数 Web 后端 | 分析/复杂业务 | 缓存/热数据 |

选型矩阵：电商/CMS → MySQL 或 PostgreSQL；实时计数/排行榜/限流 → Redis；二者通常**组合使用**：PostgreSQL 存源数据，Redis 扛热点读。本文以"MySQL + PostgreSQL + Redis 三件套"演示完整链路，覆盖缓存层、关系层与高可用。

## VPS 配置与磁盘基线建议

数据库对磁盘 I/O 极敏感，千万别和系统盘抢机械盘或低 IOPS 云盘。

| 场景 | CPU | 内存 | 磁盘 | 说明 |
|------|-----|------|------|------|
| 个人小站 | 1~2 核 | 2~4 GB | 20GB SSD | 数据量 < 1GB |
| 中型业务 | 2~4 核 | 4~8 GB | 40GB NVMe | 缓存够用 |
| 高并发 | 4+ 核 | 8~16 GB | 80GB NVMe + 独立数据盘 | 读写分离 |
| 主从 HA | 2 台同配 | 同上 | 独立盘 | 一主一从 |

关键原则：**给数据库独立数据盘**（附加盘），并优先 NVMe；`innodb_buffer_pool_size` / `shared_buffers` 通常设为可用内存的 60~70%，让热数据尽量待在内存。内存比 CPU 更重要——库是内存与磁盘的游戏，CPU 往往在瓶颈到来前还很闲。

## MySQL 单机部署与参数调优

```bash
sudo apt update && sudo apt install -y mysql-server
sudo mysql_secure_installation
```

核心调优（`/etc/mysql/mysql.conf.d/mysqld.cnf`，4GB 内存示例）：

```ini
[mysqld]
innodb_buffer_pool_size = 2G
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit = 2   # 性能/安全折中；金融级改 1
sync_binlog = 1
max_connections = 300
slow_query_log = 1
long_query_time = 1
```

`innodb_flush_log_at_trx_commit=2` + `sync_binlog=1` 是"丢最多 1 秒、性能尚可"的折中；若每笔都必须不丢（支付），两者都设 1，代价是写吞吐下降。`systemctl restart mysql` 后验证：`mysql -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"`。

## PostgreSQL 单机部署与调优

```bash
sudo apt install -y postgresql postgresql-contrib
sudo -u postgres psql -c "ALTER USER postgres PASSWORD '强密码';"
```

`/etc/postgresql/16/main/conf.d/tune.conf`：

```ini
shared_buffers = 2GB
effective_cache_size = 4GB
work_mem = 32MB
maintenance_work_mem = 512MB
wal_level = replica
max_wal_senders = 10
checkpoint_completion_target = 0.9
```

`SELECT pg_reload_conf();` 生效。PostgreSQL 默认更"严谨"（数据类型、约束强），适合对一致性要求高的业务；`work_mem` 控制排序/哈希内存，复杂查询多可适当上调但别乘连接数爆内存。

## Redis 部署与持久化策略

```bash
sudo apt install -y redis-server
```

`/etc/redis/redis.conf` 关键项：

```ini
maxmemory 1gb
maxmemory-policy allkeys-lru
appendonly yes          # AOF 持久化，防重启丢数据
appendfsync everysec    # 每秒刷盘，性能/安全折中
save 900 1              # RDB 快照兜底
```

持久化抉择：缓存型（可丢）→ 关 AOF 只 RDB；需持久（会话/限流计数）→ 开 AOF `everysec`。**永远不要把 Redis 当唯一数据源存钱**——它重启、OOM、主从切换都可能丢最近写入。

## 主从复制：MySQL 实战

一主一从，主写从读，既分担读压力又提供容灾。

主库 `my.cnf`：`server-id=1; log_bin=mysql-bin; binlog_format=ROW`。

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl密码';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
SHOW MASTER STATUS;   -- 记下 File/Position
```

从库：

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='主库IP', SOURCE_USER='repl',
  SOURCE_PASSWORD='repl密码',
  SOURCE_LOG_FILE='mysql-bin.000001', SOURCE_LOG_POS=154;
START REPLICA;
SHOW REPLICA STATUS\G   -- 看 Seconds_Behind_Source=0
```

读写分离：应用写主、读从，读 QPS 可线性扩展。监控 `Seconds_Behind_Source` 防止从库严重滞后导致读到旧数据。

## PostgreSQL 流复制 standby

主库已开 `wal_level=replica`，创建复制槽并配置 `pg_hba.conf` 允许从库：

```sql
SELECT pg_create_physical_replication_slot('standby1');
```

从库用 `pg_basebackup` 拉全量再启流复制：

```bash
pg_basebackup -h 主库IP -D /var/lib/postgresql/16/main -U repl -Fp -Xs -R -P
systemctl start postgresql
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"  -- 应返回 t
```

## Redis 主从 + 哨兵

```ini
# 从节点 redis.conf
replicaof 主库IP 6379
```

哨兵（3 节点）自动故障转移：

```ini
sentinel monitor mymaster 主库IP 6379 2
sentinel down-after-milliseconds mymaster 5000
```

哨兵在 master 宕机时把某个 replica 提升为新 master，客户端经哨兵发现新主，实现高可用。

## 高可用方案对比：Keepalived / Orchestrator / Patroni

| 方案 | 适用 | 切换方式 | 复杂度 | 说明 |
|------|------|----------|--------|------|
| Keepalived + VIP | MySQL/PG 简单 HA | VIP 漂移 | 低 | 需共享 VIP，云上受限 |
| Orchestrator | MySQL | 自动选主 | 中 | 拓扑管理强，防脑裂 |
| Patroni | PostgreSQL | 自动选主（etcd） | 中高 | PG HA 事实标准 |
| 云 RDS | 不挑 | 厂商托管 | 低 | 最省心但贵 |

中小团队建议：MySQL 用 Orchestrator，PG 用 Patroni + etcd；纯演示可用 Keepalived。核心是**防脑裂（split-brain）**——两主同时写会数据撕裂，必须用 fencing/quorum（奇数哨兵/etcd 节点）。

## 连接池与性能压测

直连数据库在高并发下会被连接数打爆，务必加连接池（MySQL 用 ProxySQL/MySQL Router，PG 用 PgBouncer，Redis 客户端自带池）。连接池把"应用几百连接"复用成"数据库几十连接"，是并发的命门。

```bash
sysbench oltp_read_only --db-driver=mysql --tables=10 --table-size=100000 \
  --threads=16 --time=60 run
redis-benchmark -h 127.0.0.1 -p 6379 -t set,get -n 100000 -c 50
```

压测目标：找出 QPS 拐点与 p99 延迟，据此定连接池大小与是否需要读写分离。别只看平均值，p99/p999 才是用户体感。

## 数据库安全加固

1. **最小权限**：应用账号只给所需库表的 SELECT/INSERT/UPDATE，禁止 ALL PRIVILEGES。
2. **网络隔离**：库只监听内网/Unix socket，防火墙仅放行应用机 IP；绝不 `bind 0.0.0.0` + 弱密码暴露公网。
3. **防注入**：应用层用参数化查询（PreparedStatement），不拼 SQL 字符串。
4. **强密码 + 改默认端口**：减少被扫爆概率（非根本安全，但降噪）。
5. **定期打补丁**：`apt upgrade` 跟进安全更新。

```bash
# 仅允许应用机访问 3306
ufw allow from 10.0.0.5 to any port 3306
ufw deny 3306
```

## 监控指标与告警

| 指标 | 健康线 | 告警阈值 | 工具 |
|------|--------|----------|------|
| 连接数 | < 80% 上限 | > 90% | Prometheus exporter |
| 慢查询数/分 | 0~个位数 | 突增 | slow log |
| 主从延迟 | < 1s | > 30s | replica status |
| 缓冲命中率 | > 99% | < 95% | buffer pool 统计 |
| 磁盘使用 | < 70% | > 85% | node exporter |
| QPS | 基线平稳 | 暴跌/暴涨 | 内置状态 |

用 `mysqld_exporter` / `postgres_exporter` / `redis_exporter` 接入 Prometheus + Grafana（参见系列监控篇），把上述指标做成看板，异常自动通知。

## 慢查询与索引优化实战

慢查询是性能头号杀手。先抓：

```sql
-- MySQL：看最慢语句
SELECT * FROM sys.statement_analysis ORDER BY avg_latency DESC LIMIT 10;
-- PostgreSQL
SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;
```

优化套路：缺索引就加（`CREATE INDEX idx_user_status ON orders(user_id, status)`）；大表分页用游标/延迟关联；避免 `SELECT *` 和 `LIKE '%x%'` 前缀模糊；热点行考虑拆列或进 Redis。索引不是越多越好——写多读少的表，索引会拖慢写入。

## 容量规划与扩容

- 数据量月增估算：当前行数 × 单行字节 × 月增速 × 保留期 = 所需盘。
- 内存：buffer pool 应能装下热数据集；热数据 > 内存则需更多内存或加从库分担读。
- 扩容路径：垂直（升配）→ 读写分离（加从）→ 分库分表（ShardingSphere）/ PG 分区表。过早分片是过度设计，先垂直+读分离通常够用到中大型。

## 故障诊断速查

```bash
# 连接卡？看当前连接与状态
mysql -e "SHOW PROCESSLIST;" | grep -i 'lock\|sleep' | head
# 磁盘满？定位大表
du -sh /var/lib/mysql/* 2>/dev/null | sort -h | tail
# 复制断了？看错误
mysql -e "SHOW REPLICA STATUS\G" | grep -i 'error\|behind'
# PG 锁等待
sudo -u postgres psql -c "SELECT * FROM pg_locks WHERE NOT granted;"
```

## 备份与 PITR（按时间点恢复）

逻辑 dump 只能回到"备份时刻"；要精确到"删库前 1 秒"，需 binlog/WAL 做 PITR。

```bash
mysqldump -u root -p --single-transaction --all-databases > full.sql
mysqlbinlog --stop-datetime="2026-09-05 14:00:00" mysql-bin.0000* > rollforward.sql
# PostgreSQL WAL 归档恢复：restore_command 指向归档，recovery_target_time 设定
```

PITR 是数据库备份的"终极保险"——配合 [第 16 篇备份容灾](../vps-review-20260416) 的异地副本，形成完整数据保护闭环。库级备份务必验证可恢复，否则等于没备。

## 故障切换演练

每月模拟一次主库宕机：`systemctl stop mysql`（主）→ 确认从库提升/应用切到新主 → 恢复旧主为从 → 校验数据一致。演练记录归档，避免真故障手忙脚乱。演练要包含"应用是否自动重连新主"这一环，否则切换成功也白搭。

## 版本升级与零停机迁移

小版本升级（如 MySQL 8.0.x → 8.0.y）通常原地 `apt upgrade` 即可；大版本（8.0 → 8.4）建议走逻辑迁移：在目标新版本建从库 → 追平 → 切主 → 下线旧主，避免长停机。PG 大版本必须用 `pg_dump`/逻辑复制或 `pg_upgrade`，且注意扩展兼容性。迁移黄金法则：先在从库验证应用能跑，再切流量，并保留旧主 24 小时作回滚。

```bash
# 逻辑复制搭新版本从库（PG），追平后停旧主、提升新主
pg_dump -h 旧主 -F c 库名 | pg_restore -h 新主 -d 库名
```

## 异地多活与读写分离架构示意

中小规模推荐一地主从 + 异地只读副本即可，不要一上来追求多活（多写冲突极难）：

```
        写 ──► [主库 香港]
                   │ 复制
        读 ──► [从库 香港]  [从库 日本 异地只读]  [Redis 缓存层]
```

应用层用中间件（MySQL Router / PgBouncer + 自研路由）把写打主、读打从；异地只读副本承接海外读，降低跨区延迟。真正的多活（双写）只在多区域强合规场景才上，且需解决冲突合并，对新手是深水区。

## 推荐 VPS

数据库需要稳定线路、低延迟与独立 NVMe 盘，一台靠谱的 VPS 是底座。

### ⭐ VPSVIP（强烈推荐）

**官网**：https://vpsvip.net

| 项目 | 内容 |
|------|------|
| 机房 | 香港 / 日本 / 美国 / 新加坡 / 韩国 |
| 线路 | CN2 / 优化 / BGP 线路 |
| 配置 | NVMe 机型适合做数据库节点 |
| 特点 | 亚太优化，主从跨机房延迟低 |
| 售后 | 7×24 中文客服 |
| 支付 | 支付宝 / 微信 / 加密货币 |

**为什么数据库场景推荐 VPSVIP？**

1. **NVMe 可选**：IOPS 高，事务提交快，慢查询显著减少。
2. **多机房**：主从可放不同机房，天然容灾。
3. **线路稳**：跨节点复制不抖，主从延迟可控。
4. **独立附加盘**：数据盘与系统盘分离，安全和性能双收益。
5. **中文客服**：HA 排障有后援。

### 其他可选

| 服务商 | 特点 | 官网 |
|--------|------|------|
| 腾讯云 | 云数据库 TencentDB 托管省心 | cloud.tencent.com |
| 阿里云 | RDS/PolarDB 成熟 | aliyun.com |
| Vultr | 高频计算实例适合库 | vultr.com |
| DigitalOcean | Managed DB 易用 | digitalocean.com |

## 相关资源

- https://vpsvip.net — VPSVIP 官网（本篇主推，NVMe 数据库节点）
- https://clashvip.net — ClashVIP 机场（跨区管理数据库时的稳定通道）
- https://nav.clashvip.net — 导航站，聚合机场 / VPS / 工具入口
- https://clashhub.net — ClashHub 客户端与配置资源
- https://bbs.clashhub.net — 社区论坛，数据库运维经验交流
- https://clash-for-windows.net — Clash for Windows 使用与下载
- https://www.bt.cn — 宝塔面板（图形化装库与备份）
- https://www.v2ex.com — V2EX 技术社区

## 常见问题

### Q：1 核 1GB 能跑 MySQL 吗？

A：能跑但极容易 OOM。最低建议 2 核 4GB，并把 buffer pool 调小，避免和系统在内存上打架。

### Q：主从延迟高怎么办？

A：检查网络（跨机房延迟）、从库硬件、大事务；可改用并行复制（MySQL `replica_parallel_workers`、PG 并行 apply）。

### Q：Redis 会丢数据吗？

A：开 AOF `everysec` 最多丢 1 秒；`always` 不丢但慢；纯 RDB 可能丢最近快照后的数据。按业务选。

### Q：要不要上云 RDS？

A：预算充足、不想运维 → 直接 RDS。想省钱且愿意学 → 本文自管方案月成本可低一个数量级。

### Q：PITR 必须做吗？

A：金融/订单类必须；个人博客逻辑 dump 足够。但凡"删了要命"，就上 binlog/WAL 归档。

### Q：索引越多越好？

A：不是。索引加速读但拖慢写、占空间。写多读少表要克制，定期用慢查询分析清理冗余索引。

## 免责声明

1. 本仓库仅提供信息参考，不构成任何商业承诺。
2. 请遵守当地法律法规使用 VPS 与数据库服务。
3. HA 方案需结合业务 SLA 定制，本文为通用实践。
4. 保护好数据库账号与备份文件。

## 许可证

MIT License

---
更新时间：2026-09-05 · 专题：VPS 数据库选型与高可用部署实战
