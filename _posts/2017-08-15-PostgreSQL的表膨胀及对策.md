#PostgreSQL的表膨胀及对策

PostgreSQL的MVCC机制在数据更新时会产生dead元组，这些dead元组通过后台的autovacuum进程清理。一般情况下autovacuum可以工作的不错，但以下情况下，dead元组可能会不断堆积。

1. autovacuum清理速度赶不上dead元组产生速度
2. 由于以下因素导致dead元组无法被回收
	- 存在长事务
	- 存在未处理的未决事务
	- 存在断开的复制槽


## 检查表膨胀

方法1：使用`pg_stat_all_tables`系统表

	SELECT
	    schemaname||'.'||relname,
	    n_dead_tup,
	    n_live_tup,
	    round(n_dead_tup * 100 / (n_live_tup + n_dead_tup),2) AS dead_tup_ratio
	FROM
	    pg_stat_all_tables
	WHERE
	    n_dead_tup >= 10000
	ORDER BY dead_tup_ratio DESC
	LIMIT 10;

方法2:使用`pg_bloat_check`

	`pg_bloat_check`会进行全表扫描，比`pg_stat_all_tables`准确，但比较慢对系统性能冲击也较大，不建议使用。


## 预防表膨胀
1. 调整autovacuum相关参数，加快垃圾回收速度
	`autovacuum_vacuum_cost_limit`的默认值过小，尤其在SSD机器上，可以适当调大。

		autovacuum_vacuum_cost_limit = 4000

2. 监视以下可能导致dead元组无法被回收的状况
	- 长事务
	- 未决事务
	- 断开的复制槽

3. 强制回收

	设置`old_snapshot_threshold`参数，强制删除为过老的事务快照保留的dead元组。

		old_snapshot_threshold = 12h

	`old_snapshot_threshold`参数不能在线修改，如果要运行更长的事务（大数据量导出），可以临时在备机上设置`max_standby_streaming_delay = -1`，然后在备机执行长事务(会带来主备延迟)。


4. 杀死长事务

	设置可以部分避免长事务的参数

		idle_in_transaction_session_timeout = 60s
		lock_timeout = 70s

## 相关代码

	vacuum()
	  ->vacuum_rel()
	      ->vacuum_set_xid_limits()
	          ->GetOldestXmin()
	              找出以下最小的事务ID，大于该事务ID的事务删除的tuple将不回收
	              - backend_xid，所有后端进程的当前事务ID的最小值
	              - backend_xmin，所有后端进程的事务启动时的事务快照中最小事务的最小值
	              - replication_slot_xmin，所有复制槽中最小的xmin
	              - replication_slot_catalog_xmin，所有复制槽中最小的catalog_xmin
	          ->TransactionIdLimitedForOldSnapshots()
					如果设置了old_snapshot_threshold，则比backend_xid和old_snapshot_threshold->xmin都老的dead元组也可以被回收

## 参考

-[PostgreSQL 9.6 快照过旧 - 源码浅析](https://yq.aliyun.com/articles/61292)

