## [中文说明](https://github.com/wuzhc/gopuser/blob/master/README-zh.md)

> Gopusher can be used as the access layer of websocket connection, responsible for connection management and message push. You can use it in instant chat or message push system


## Introduction 
- Using `golang`, high performance, single machine can support millions of connections
- Message push uses `grpc + protobuf `communication protocol, supports `HTTP` and `RPC` push, and can be used in various programming languages
- Multiple message processors can be set, and concurrent execution of multiple processors can ensure that messages can be pushed to the client in time
- Message middleware `NSQ` can support the accumulation of a large number of messages and prevent the system from being overwhelmed by a large number of push messages
- The message push `TPS`  is about 10000 (op/s), the environment is 8g memory, 4-core CPU, and the message content length is 11 bytes (in fact, it will be better, because the pressure test and the pressure test are conducted on the same machine)
- Use `nginx` or `LVS` for load balancing and  `etcd + confd` for highly available registration and discovery services

## Framework
`Gopusher` has five modules:
- Queue: queue module for message storage
- Service: service module, used to process API
- Socket: connection management module
- Config: configuration module
- Web: web module

### Single  architecture
![](https://gitee.com/wuzhc123/zcnote/raw/master/images/project/gopusher_2.png)
### Distributed architecture
![](https://gitee.com/wuzhc123/zcnote/raw/master/images/project/gopusher.png)

## How to use
### How to establish a connection
```javascript
var ws = new WebSocket("ws://127.0.0.1:8080/ws");
ws.onopen = function()
{
	// Bind card_id and app_id to connecton
    ws.send(JSON.stringify({"event":"join","data":{"app_id":"alibaba","card_id":"mayun"}}));
};

ws.onmessage = function (evt) 
{ 
    var msg = evt.data;
    console.log(msg)
};

ws.onclose = function()
{ 
    console.log("closed.")
};
```
- When establishing a conenction, you need to bind a `card_id` to the connection to identify who the connection is, and the `card_id` can be a user or a group
- In `gopusher`, the same `card_id` can be classified into the same group. For example, when `card_id` is a user, the user may open multiple tabs in the browser, at this time, the user has multiple websocket connections, and these connections will be grouped into the same group; other, when the user establishes a connection on the mobile terminal, the user will also be Belong to the same group. If you need to distinguish between different terminals or different applications, you can set `app_id`
### The relationship of `card_id`, group and connection is as follows：
![](https://gitee.com/wuzhc123/zcnote/raw/master/images/project/gopusher_card_id.png)

### How to push message
- http mode
The default port is 8081, which can be modified in the `gatewayaddr` option of the `config. Ini` configuration file
```bash
# Push the message to two `card_id`, which can be users or groups
curl -X POST -k http://127.0.0.1:8081/push -d '{"from":"xxx","to":["wuzhc_1","wuzhc_2"], "content":"hellwo world"}'
```

- rpc mode
Refer to：service/service_test.go
```go
package main

import (
	"context"
	pb "github.com/wuzhc/gopusher/proto"
	"google.golang.org/grpc"
	"log"
)

func main() {
	conn, err := grpc.Dial("127.0.0.1:9002", grpc.WithInsecure())
	if err != nil {
		t.Fatal(err)
	}
	defer conn.Close()

	// Contact the server and print out its response.
	c := pb.NewRpcClient(conn)
	r, err := c.Push(context.Background(), &pb.PushRequest{
		From:    "xxx",
		To:      []string{"wuzhc_1", "wuzhc_2"},
		Content: "hello world",
	})
	if err != nil {
		log.Fatalln(err)
	} else {
		log.Printf("recv message:%s\n", r.Message)
	}
}
```

## Distributed deployment
- Dependent components
```
https://github.com/nsqio/nsq
https://github.com/etcd-io/etcd
https://github.com/kelseyhightower/confd
```
- Start
```bash
# start nsq
nsqlookupd 
nsq options.go --lookupd-tcp-address=127.0.0.1:4160 -tcp-address=0.0.0.0:4152 -http-address=0.0.0.0:4153

# start nginx
nginx -c /usr/local/nginx/conf/nginx.conf

# start etcd
etcd

# start confd
confd -watch -backend etcdv3 -node http://127.0.0.1:2379
```
- Configuration of each component
[Nginx configuration](https://github.com/wuzhc/zcnote/blob/master/%E9%A1%B9%E7%9B%AE/%E6%8E%A8%E9%80%81%E7%B3%BB%E7%BB%9F2.0/nginx%E9%85%8D%E7%BD%AE.md)
[Confd configuration](https://github.com/wuzhc/zcnote/blob/master/%E9%A1%B9%E7%9B%AE/%E6%8E%A8%E9%80%81%E7%B3%BB%E7%BB%9F2.0/confd%E9%85%8D%E7%BD%AE.md)
