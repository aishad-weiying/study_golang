## go 语言与 redis 的交互二

1. 安装需要的包
```go
go get -u github.com/go-redis/redis
// 如果报错
git clone https://github.com/open-telemetry/opentelemetry-go $GOPATH/src/go.opentelemetry.io/otel
```

2. 普通连接
```go
package main

import (
	"context"
	"fmt"

	"github.com/go-redis/redis"
)

var rdb *redis.Client

func connRedis() error {
	var ctx = context.Background()
	rdb = redis.NewClient(&redis.Options{
		Addr:     "172.19.36.48:6379",
		Password: "admin123",
		DB:       0,// 使用默认的库
	})
	_, err := rdb.Ping(ctx).Result()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := connRedis()
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println("redis连接成功")
}
```

3. 连接 redis 哨兵
```go
func initClient() (err error) {
	var ctx = context.Background()
	rdb := redis.NewFailoverClient(&redis.FailoverOptions{
		MasterName:    "master",
		SentinelAddrs: []string{"x.x.x.x:26379", "xx.xx.xx.xx:26379", "xxx.xxx.xxx.xxx:26379"},
	})
	_, err = rdb.Ping(ctx).Result()
	if err != nil {
		return err
	}
	return nil
}
```

4. 连接 redis 集群
```go
func initClient()(err error){
    var ctx = context.Background()
	rdb := redis.NewClusterClient(&redis.ClusterOptions{
		Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},
	})
	_, err = rdb.Ping(ctx).Result()
	if err != nil {
		return err
	}
	return nil
}
```

## 基本使用

1. set 的使用
```go
func main() {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "172.19.36.48:6379",
		Password: "admin123",
		DB:       0,
	})
	ctx := context.Background()

	_, err := rdb.Ping(ctx).Result()
	if err != nil {
		log.Println("redis连接失败:", err)
		return
	}
	log.Println("redis连接成功")
    // 返回值是成功与否的状态
	status := rdb.Set(ctx, "sore", 100, 0)
	fmt.Println(status)
    //
	err = rdb.Set(ctx, "age", 22, 0).Err()
	if err != nil {
		log.Println("redis 添加键值失败:", err)
		return
	}
}
```

2. get
```go
func main() {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "172.19.36.48:6379",
		Password: "admin123",
		DB:       0,
	})
	ctx := context.Background()

	_, err := rdb.Ping(ctx).Result()
	if err != nil {
		log.Println("redis连接失败:", err)
		return
	}
	log.Println("redis连接成功")

    // 获取键值
	val, err := rdb.Get(ctx, "sore").Result()
	if err != nil {
		log.Println("redis 获取键值失败:", err)
		return
	}
	log.Println(val)
	val2, err := rdb.Get(ctx, "name").Result()
	if err == redis.Nil {
		fmt.Println("name does not exist")
	} else if err != nil {
		fmt.Printf("get name failed, err:%v\n", err)
		return
	} else {
		fmt.Println("name", val2)
	}
}
```

3. 集合
```go
func main() {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "172.19.36.48:6379",
		Password: "admin123",
		DB:       0,
	})
	ctx := context.Background()

	_, err := rdb.Ping(ctx).Result()
	if err != nil {
		log.Println("redis连接失败:", err)
		return
	}
	log.Println("redis连接成功")
	// 创建一个key
	key := "rank"
	// 元素
	language := []*redis.Z{
		&redis.Z{Score: 90.0, Member: "Golang"},
		&redis.Z{Score: 98.0, Member: "Java"},
		&redis.Z{Score: 95.0, Member: "Python"},
		&redis.Z{Score: 97.0, Member: "JavaScript"},
		&redis.Z{Score: 99.0, Member: "C/C++"},
	}
	// zadd 追加数据,language... 表示把language下面的每个数据逐个追加
	num, err := rdb.ZAdd(ctx, key, language...).Result()
	if err != nil {
		log.Println("redis 添加集合失败:", err)
		return
	}
	log.Println("添加的数据条目为:", num)
	// 把golang的分数+10
	newscore, err := rdb.ZIncrBy(ctx, key, 10.0, "Golang").Result()
	if err != nil {
		log.Println("更新集合值失败:", err)
		return
	}
	log.Println("新的分数为:", newscore)
	// 取分数前三
	ret, err := rdb.ZRevRangeWithScores(ctx, key, 0, 2).Result()
	if err != nil {
		log.Println("获取前三失败:", err)
		return
	}
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}
	// 取出分数为95-100之间的
	op := &redis.ZRangeBy{
		Min: "95",
		Max: "100",
	}
	ret, err = rdb.ZRangeByScoreWithScores(ctx,key, op).Result()
	if err != nil {
		fmt.Printf("zrangebyscore failed, err:%v\n", err)
		return
	}
	fmt.Println("分数在95-100之间的有:")
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}
}
```