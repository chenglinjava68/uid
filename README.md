# 高性能分布式自增id生成器lid
先看下测试结果：

```
goos: darwin
goarch: amd64
pkg: goim/public/lib/lid
BenchmarkLeafKey-4   	 2000000	      1081 ns/op
PASS
```

步长设置为1000.缓冲池大小设为1000，每秒可以达到近百万次的生成量，其思想借鉴了[Leaf——美团点评分布式ID生成系统](https://tech.meituan.com/MT_Leaf.html)的Leaf-segment数据库双buffer优化方案，其实他的核心思想是，每次从数据库拿取一个号段，用完了，再去数据库拿，当用尽去数据库拿的时候，会有一小会的阻塞，对这一情况做了一些优化。

刚开始实现的时候，和美团的方案一样，利用两个buffer，Leaf服务内部有两个号段缓存区segment。当前号段已下发10%时，如果下一个号段未更新，则另启一个更新线程去更新下一个号段。当前号段全部下发完后，如果下个号段准备好了则切换到下个号段为当前segment接着下发，循环往复。

最后想了想，其实没必要这么复杂，用一个channal,一边起一个goroutine，先从数据库拿取一个号段，然后生成id放到channel里面，如果号段用尽，再从数据库里面取，如此往复，当channel里面满时，goroutine会阻塞。一边用的时候从里面拿就行。

贴出代码：

```go
package uid

import (
	"database/sql"
	"errors"
	"log"
	"time"
)

type logger interface {
	Error(error)
}

// Logger Log接口，如果设置了Logger，就使用Logger打印日志，如果没有设置，就使用内置库log打印日志
var Logger logger

// ErrTimeOut 获取uid超时错误
var ErrTimeOut = errors.New("get uid timeout")

type Uid struct {
	db         *sql.DB    // 数据库连接
	businessId string     // 业务id
	ch         chan int64 // id缓冲池
	min, max   int64      // id段最小值，最大值
}

// NewUid 创建一个Uid;len：缓冲池大小()
// db:数据库连接
// businessId：业务id
// len：缓冲池大小(长度可控制缓存中剩下多少id时，去DB中加载)
func NewUid(db *sql.DB, businessId string, len int) (*Uid, error) {
	lid := Uid{
		db:         db,
		businessId: businessId,
		ch:         make(chan int64, len),
	}
	go lid.productId()
	return &lid, nil
}

// Get 获取自增id,当发生超时，返回错误，避免大量请求阻塞，服务器崩溃
func (u *Uid) Get() (int64, error) {
	select {
	case <-time.After(1 * time.Second):
		return 0, ErrTimeOut
	case uid := <-u.ch:
		return uid, nil
	}
}

// productId 生产id，当ch达到最大容量时，这个方法会阻塞，直到ch中的id被消费
func (u *Uid) productId() {
	u.reLoad()

	for {
		if u.min >= u.max {
			u.reLoad()
		}

		u.min++
		u.ch <- u.min
	}
}

// reLoad 在数据库获取id段，如果失败，会每隔一秒尝试一次
func (u *Uid) reLoad() error {
	var err error
	for {
		err = u.getFromDB()
		if err == nil {
			return nil
		}

		// 数据库发生异常，等待一秒之后再次进行尝试
		if Logger != nil {
			Logger.Error(err)
		} else {
			log.Println(err)
		}
		time.Sleep(time.Second)
	}
}

// getFromDB 从数据库获取id段
func (u *Uid) getFromDB() error {
	var (
		maxId int64
		step  int64
	)

	tx, err := u.db.Begin()
	if err != nil {
		return err
	}
	defer tx.Rollback()

	row := tx.QueryRow("SELECT max_id,step FROM uid WHERE business_id = ? FOR UPDATE", u.businessId)
	err = row.Scan(&maxId, &step)
	if err != nil {
		return err
	}

	_, err = tx.Exec("UPDATE uid SET max_id = ? WHERE business_id = ?", maxId+step, u.businessId)
	if err != nil {
		return err
	}
	err = tx.Commit()
	if err != nil {
		return err
	}

	u.min = maxId
	u.max = maxId + step
	return nil
}
```
sql语句
```sql
CREATE TABLE `uid` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `business_id` varchar(128) COLLATE utf8mb4_bin DEFAULT NULL COMMENT '业务id',
  `max_id` bigint(20) unsigned DEFAULT NULL COMMENT '最大id',
  `step` int(10) unsigned DEFAULT NULL COMMENT '步长',
  `description` varchar(255) COLLATE utf8mb4_bin DEFAULT NULL COMMENT '描述',
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_business_id` (`business_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin COMMENT='分布式自增主键';
```



