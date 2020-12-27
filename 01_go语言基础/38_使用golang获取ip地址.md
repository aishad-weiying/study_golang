## 使用 go 语言获取本机的 ip 地址
```go
package main

import (
	"fmt"
	"net"
	"strings"
)

func GetOutboundIP()(ip string,err error)  {
	conn ,err := net.Dial("udp","8.8.8.8:80") // 不回去真正的连接这个 ip
	if err != nil {
		return
	}
	
	defer conn.Close()
	
	localAddr := conn.LocalAddr().(*net.UDPAddr)
	fmt.Println(localAddr.String())
	ip = strings.Split(localAddr.IP.String(),":")[0]
	return
}

func main()  {
	ip ,err := GetOutboundIP()
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(ip)
}
```