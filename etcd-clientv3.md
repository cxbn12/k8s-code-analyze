## clientv3是官方提供的etcd v3的go客户端实现

### clientv3用法详解

#### 创建client
要访问etcd第一件事就是创建client，它需要传入一个Config配置，这里传了2个选项：
* Endpoints：etcd的多个节点服务地址，因为我是单点测试，所以只传1个。
* DialTimeout：创建client的首次连接超时，这里传了5秒，如果5秒都没有连接成功就会返回err；值得注意的是，一旦client创建成功，我们就不用再关心后续底层连接的状态了，client内部会重连。

代码示例如下:
```
cli, err := clientv3.New(clientv3.Config{
   Endpoints:   []string{"localhost:2378"},
   DialTimeout: 5 * time.Second,
})
```
创建的client结构如下：
```
type Client struct {
    Cluster
    KV
    Lease
    Watcher
    Auth
    Maintenance

    // Username is a user name for authentication.
    Username string
    // Password is a password for authentication.
    Password string
    // contains filtered or unexported fields
}
```
Cluster：向集群里增加etcd服务端节点之类，属于管理员操作。
KV：我们主要使用的功能，即操作K-V。
Lease：租约相关操作，比如申请一个TTL=10秒的租约。
Watcher：观察订阅，从而监听最新的数据变化。
Auth：管理etcd的用户和权限，属于管理员操作。
Maintenance：维护etcd，比如主动迁移etcd的leader节点，属于管理员操作。
#### 获取KV对象
实际上client.KV是一个interface，提供了关于k-v操作的所有方法：
```
type KV interface {
	// Put puts a key-value pair into etcd.
	// Note that key,value can be plain bytes array and string is
	// an immutable representation of that bytes array.
	// To get a string of bytes, do string([]byte{0x10, 0x20}).
	Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)

	// Get retrieves keys.
	// By default, Get will return the value for "key", if any.
	// When passed WithRange(end), Get will return the keys in the range [key, end).
	// When passed WithFromKey(), Get returns keys greater than or equal to key.
	// When passed WithRev(rev) with rev > 0, Get retrieves keys at the given revision;
	// if the required revision is compacted, the request will fail with ErrCompacted .
	// When passed WithLimit(limit), the number of returned keys is bounded by limit.
	// When passed WithSort(), the keys will be sorted.
	Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)

	// Delete deletes a key, or optionally using WithRange(end), [key, end).
	Delete(ctx context.Context, key string, opts ...OpOption) (*DeleteResponse, error)

	// Compact compacts etcd KV history before the given rev.
	Compact(ctx context.Context, rev int64, opts ...CompactOption) (*CompactResponse, error)

	// Do applies a single Op on KV without a transaction.
	// Do is useful when creating arbitrary operations to be issued at a
	// later time; the user can range over the operations, calling Do to
	// execute them. Get/Put/Delete, on the other hand, are best suited
	// for when the operation should be issued at the time of declaration.
	Do(ctx context.Context, op Op) (OpResponse, error)

	// Txn creates a transaction.
	Txn(ctx context.Context) Txn
}
```
但是我们一般并不是直接获取client.KV来使用，而是通过一个方法来获得一个经过装饰的KV实现（内置错误重试机制的高级KV）来操作ETCD中的数据：
```
kv := clientv3.NewKV(cli)
```
##### Put操作
函数原型为：
```
	// Put puts a key-value pair into etcd.
	// Note that key,value can be plain bytes array and string is
	// an immutable representation of that bytes array.
	// To get a string of bytes, do string([]byte{0x10, 0x20}).
	Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)
```
除了我们传递的参数，还支持一个可变参数，主要是传递一些控制项来影响Put的行为，例如可以携带一个lease ID来支持key过期，这个后面再说。

上述Put操作返回的是PutResponse，不同的KV操作对应不同的response结构。
```
type PutResponse struct {
	Header *ResponseHeader `protobuf:"bytes,1,opt,name=header" json:"header,omitempty"`
	// if prev_kv is set in the request, the previous key-value pair will be returned.
	PrevKv *mvccpb.KeyValue `protobuf:"bytes,2,opt,name=prev_kv,json=prevKv" json:"prev_kv,omitempty"`
}
```
Header里保存的主要是本次更新的revision信息，而PrevKv可以返回Put覆盖之前的value是什么
```
	// 再写一个孩子
	kv.Put(context.TODO(),"/test/b", "another")

	// 再写一个同前缀的干扰项
	kv.Put(context.TODO(), "/testxxx", "干扰")
```
##### Get操作
函数原型为:
```
	// Get retrieves keys.
	// By default, Get will return the value for "key", if any.
	// When passed WithRange(end), Get will return the keys in the range [key, end).
	// When passed WithFromKey(), Get returns keys greater than or equal to key.
	// When passed WithRev(rev) with rev > 0, Get retrieves keys at the given revision;
	// if the required revision is compacted, the request will fail with ErrCompacted .
	// When passed WithLimit(limit), the number of returned keys is bounded by limit.
	// When passed WithSort(), the keys will be sorted.
	Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)
```
和Put类似，函数注释里提示我们可以传递一些控制参数来影响Get的行为，比如：WithFromKey表示读取从参数key开始递增的所有key，而不是读取单个key。

在上面的例子中，我没有传递opOption，所以就是获取key=/test/a的最新版本数据。

这里err并不能反馈出key是否存在（只能反馈出本次操作因为各种原因异常了），我们需要通过GetResponse（实际上是pb.RangeResponse）判断key是否存在：
```
type RangeResponse struct {
	Header *ResponseHeader `protobuf:"bytes,1,opt,name=header" json:"header,omitempty"`
	// kvs is the list of key-value pairs matched by the range request.
	// kvs is empty when count is requested.
	Kvs []*mvccpb.KeyValue `protobuf:"bytes,2,rep,name=kvs" json:"kvs,omitempty"`
	// more indicates if there are more keys to return in the requested range.
	More bool `protobuf:"varint,3,opt,name=more,proto3" json:"more,omitempty"`
	// count is set to the number of keys within the range when requested.
	Count int64 `protobuf:"varint,4,opt,name=count,proto3" json:"count,omitempty"`
}
```
Kvs字段，保存了本次Get查询到的所有k-v对，因为上述例子只Get了一个单key，所以只需要判断一下len(Kvs)是否==1即可知道是否存在。
KeyValue的结构描述为：
```
type KeyValue struct {
	// key is the key in bytes. An empty key is not allowed.
	Key []byte `protobuf:"bytes,1,opt,name=key,proto3" json:"key,omitempty"`
	// create_revision is the revision of last creation on this key.
	CreateRevision int64 `protobuf:"varint,2,opt,name=create_revision,json=createRevision,proto3" json:"create_revision,omitempty"`
	// mod_revision is the revision of last modification on this key.
	ModRevision int64 `protobuf:"varint,3,opt,name=mod_revision,json=modRevision,proto3" json:"mod_revision,omitempty"`
	// version is the version of the key. A deletion resets
	// the version to zero and any modification of the key
	// increases its version.
	Version int64 `protobuf:"varint,4,opt,name=version,proto3" json:"version,omitempty"`
	// value is the value held by the key, in bytes.
	Value []byte `protobuf:"bytes,5,opt,name=value,proto3" json:"value,omitempty"`
	// lease is the ID of the lease that attached to key.
	// When the attached lease expires, the key will be deleted.
	// If lease is 0, then no lease is attached to the key.
	Lease int64 `protobuf:"varint,6,opt,name=lease,proto3" json:"lease,omitempty"`
}
```
至于RangeResponse.More和Count，当我们使用withLimit()选项进行Get时会发挥作用，相当于翻页查询。

接下来，我们通过一个特别的Get选项，获取/test目录下的所有孩子：
```
rangeResp, err := kv.Get(context.TODO(), "/test/", clientv3.WithPrefix())
```
WithPrefix()是指查找以/test/为前缀的所有key，因此可以模拟出查找子目录的效果。
我们知道etcd是一个有序的k-v存储，因此/test/为前缀的key总是顺序排列在一起。

withPrefix实际上会转化为范围查询，它根据前缀/test/生成了一个key range，[“/test/”, “/test0”)，为什么呢？因为比/大的字符是’0’，所以以/test0作为范围的末尾，就可以扫描到所有的/test/打头的key了。

在之前，我Put了一个/testxxx干扰项，因为不符合/test/前缀（注意末尾的/），所以就不会被这次Get获取到。但是，如果我查询的前缀是/test，那么/testxxx也会被扫描到，这就是etcd k-v模型导致的，编程时一定要特别注意。

##### 获取Lease对象
通过下面的代码获取Lease对象
```
lease := clientv3.NewLease(cli)
```
Lease描述如下：
```
type Lease interface {
	// Grant creates a new lease.
	Grant(ctx context.Context, ttl int64) (*LeaseGrantResponse, error)

	// Revoke revokes the given lease.
	Revoke(ctx context.Context, id LeaseID) (*LeaseRevokeResponse, error)

	// TimeToLive retrieves the lease information of the given lease ID.
	TimeToLive(ctx context.Context, id LeaseID, opts ...LeaseOption) (*LeaseTimeToLiveResponse, error)

	// Leases retrieves all leases.
	Leases(ctx context.Context) (*LeaseLeasesResponse, error)

	// KeepAlive keeps the given lease alive forever.
	KeepAlive(ctx context.Context, id LeaseID) (<-chan *LeaseKeepAliveResponse, error)

	// KeepAliveOnce renews the lease once. In most of the cases, KeepAlive
	// should be used instead of KeepAliveOnce.
	KeepAliveOnce(ctx context.Context, id LeaseID) (*LeaseKeepAliveResponse, error)

	// Close releases all resources Lease keeps for efficient communication
	// with the etcd server.
	Close() error
}
```
Lease提供了几个功能：

* Grant：分配一个租约。
* Revoke：释放一个租约。
* TimeToLive：获取剩余TTL时间。
* Leases：列举所有etcd中的租约。
* KeepAlive：自动定时的续约某个租约。
* KeepAliveOnce：为某个租约续约一次。
* Close：貌似是关闭当前客户端建立的所有租约。

###### Grant与TTL
要想实现key自动过期，首先得创建一个租约，它有10秒的TTL：
```
grantResp, err := lease.Grant(context.TODO(), 10)
```
grantResp中主要使用到了ID，也就是租约ID：
```
// LeaseGrantResponse wraps the protobuf message LeaseGrantResponse.
type LeaseGrantResponse struct {
	*pb.ResponseHeader
	ID    LeaseID
	TTL   int64
	Error string
}
```
接下来，我们用这个租约来Put一个会自动过期的Key：
```
kv.Put(context.TODO(), "/test/expireme", "gone...", clientv3.WithLease(grantResp.ID))
```

这里特别需要注意，有一种情况是在Put之前Lease已经过期了，那么这个Put操作会返回error，此时你需要重新分配Lease。

当我们实现服务注册时，需要主动给Lease进行续约，这需要调用KeepAlive/KeepAliveOnce，你可以在一个循环中定时的调用：
```
keepResp, err := lease.KeepAliveOnce(context.TODO(), grantResp.ID)
// sleep一会..
```

keepResp结构如下：
```
// LeaseKeepAliveResponse wraps the protobuf message LeaseKeepAliveResponse.
type LeaseKeepAliveResponse struct {
	*pb.ResponseHeader
	ID  LeaseID
	TTL int64
}
```
没什么特别需要用到的字段。

KeepAlive和Put一样，如果在执行之前Lease就已经过期了，那么需要重新分配Lease。Etcd并没有提供API来实现原子的Put with Lease。

### 一个完整的应用示例
```
package main

import (
	"log"
	"time"

	"golang.org/x/net/context"

	"github.com/coreos/etcd/clientv3"
)

var (
	dialTimeout    = 5 * time.Second
	requestTimeout = 2 * time.Second
	endpoints      = []string{"localhost:2379"}
)

func main() {
       //创建客户端
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   endpoints,
		DialTimeout: dialTimeout,
	})
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ch := make(chan struct{})
	//开启节点watch routine
	go func() {
		rch := cli.Watch(context.Background(), "", clientv3.WithPrefix())
		for wresp := range rch {
			for _, ev := range wresp.Events {
				log.Printf("Watch: %s %q: %q \n", ev.Type, ev.Kv.Key, ev.Kv.Value)
			}
		}
		ch <- struct{}{}
	}()
        //创建2s的lease
	resp, _ := cli.Grant(context.TODO(), 2)

	kvs := map[string]string{
		"key1": "value1",
		"key2": "value2",
		"key3": "value3",
		"key4": "value4",
	}
	//创建存储节点键值对
	for k, v := range kvs {
		cli.Put(context.TODO(), k, v, clientv3.WithLease(resp.ID))
	}
        //更新键值
	cli.Put(context.TODO(), "key2", "value2new")
	cli.Put(context.TODO(), "key3", "value3new", clientv3.WithIgnoreLease())
	resp, _ = cli.Grant(context.TODO(), 4)
	cli.Put(context.TODO(), "key4", "value4new", clientv3.WithLease(resp.ID))

	select {
	case <-ch:
	case <-time.After(5 * time.Second):
	}
}
```
输出结果为：
```
2018/05/11 13:37:48 Watch: PUT "key4": "value4"
2018/05/11 13:37:48 Watch: PUT "key1": "value1"
2018/05/11 13:37:48 Watch: PUT "key2": "value2"
2018/05/11 13:37:48 Watch: PUT "key3": "value3"
2018/05/11 13:37:48 Watch: PUT "key2": "value2new"
2018/05/11 13:37:48 Watch: PUT "key3": "value3new"
2018/05/11 13:37:48 Watch: PUT "key4": "value4new"
2018/05/11 13:37:50 Watch: DELETE "key1": ""
2018/05/11 13:37:50 Watch: DELETE "key3": ""
2018/05/11 13:37:52 Watch: DELETE "key4": ""
```


