---
author: "Narcissus"
title: "Golang实现分布式缓存"
date: "2022-08-29"
lastmod: "2022-08-29"
description: "使用Golang实现一个简单的分布式缓存系统。"
tags: ["Golang", "分布式缓存"]
categories: ["Golang"]
---

## GroupCache

### 基本介绍

groupcache是一个小巧的kv存储库，**最大的特点就是没有删除接口，**即kv键值一旦设置进去了，用户端是没有主动的手段删除这个值的，这个值将不能被用户修改，k的v不能被修改，**带来的好处就是没有覆盖更新带来的一致性问题**。

- **groupcache没有更新和删除接口，空间岂不是会越来越多？还有实用意义吗？**

groupcache没有set、update、delete接口只是让用户无法更新和删除已经缓存的内容而已，但不是说设置进去的kv要永久保存，groupcache内存通过LRU来管理内容的。

- **groupcache没有set接口，那内容是如何设置进去的呢？**

1. 初始化的时候，就需要明确当 key miss 的时候，怎么获取到内容的手段，把这个手段配置好是前提；
2. get 调用的时候，当 key miss 的时候，就会调用初始化的获取手段来获取数据，如果 hit 的话，那么就直接返回了；

- **这种只能get，不能更新key的缓存有啥用？有什么使用场景？**

比如缓存一些静态文件，就很适合groupcache这种缓存，因为key对应的value不需要变。

### group机制

#### 协商填充

固定的key由固定的节点服务，这个相当于一个绑定。

#### 热点数据多节点备份

分布式缓存系统中，一般需要从两个层面考虑热点均衡问题：

1. 大量的key是否均衡的分布到节点，**这个指请求数量的分布均衡**
2. 某些key的访问频率也是有热点的，也是不均衡的。

针对第一点，不同节点会负责特定的 key 集合的查询请求，一般来讲只要哈希算法还行， key 的分布是均衡的。

针对第二点，某些key属于热点数据而被大量访问，这会导致压力全部都在某个节点上。

**groupcache有一个热点数据自动扩展机制来解决这个问题，针对每个节点，除了会缓存本节点存在且大量访问的key之外，也会缓存那些不属于节点的(但是被频繁访问)的key。**



实现方式上：每一个缓存节点（Group）有两个缓存实体：mainCache、hotCache。mainCache主要用来存放应该存放在本缓存节点的数据。hotCache用来存放热点数据，**实现上：每次从其他缓存节点获取数据后，都有一定的概率将该数据存放到hotCache中供下一次调用，减少了HTTP通信。** mianCache和hotCache的缓存大小比例是8:1的比例。即需要驱逐旧数据的时候，判断`hotBytes > mainBytes/8` 如果成立则驱逐hotCache，否则驱逐mianCache。

> 一定概率是指：目前采用比较简单的方式实现，即`rand.Intn(10) == 0` 则将key/value存储在改group中的hotCache中。 可以使用MinutesQPS或者更好的方式来选择是否缓存热点数据。

## 概述

### 1. 实现的特性

- 单机缓存和基于HTTP的分布式缓存
- 使用最近最少访问（LRU）缓存策略
- 使用Go锁机制防止缓存穿透
- 使用一致性哈希选择节点，实现负载均衡
- 使用protobuf优化节点间二进制通信

### 2. 整体流程

![image-20211210171332461](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211210171332461.png)

每个缓存节点包含多个Group。每一个Group都有一个缓存mainCache（通过LRU缓存淘汰算法实现），一个回调函数与数据源相连接，以及一个包含HTTP通信的peers。

> 注意：peers(即HTTPPool)承担了服务端与客户端的功能，可以与group分离，控制整个缓存节点，本项目通过RegisterPeer方法将其注入到了Group中。

整体流程图如下：

![未命名文件](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6.png)

## LRU缓存淘汰策略

### 1. FIFO/LFU/LRU算法简介

#### 1.1 FIFO

先进先出，即淘汰缓存中最老的记录。该算法实现简单，用一个队列即可模拟。很多场景下，部分记录虽然最早添加但也常被访问，而不得不因为呆的时间太久而被淘汰，这类数据会被频繁添加进缓存，又被淘汰，导致缓存命中率降低。

#### 1.2 LFU

最少使用，淘汰缓存中频率最低的记录。LFU认为，如果数据过去被访问多次，那么将来被访问的频率也更高。缺点如下：

- 维护每个记录的访问次数，对内存的消耗很高
- 如果数据的访问模式发生变化，LFU需要较长时间去适应，即LFU受历史数据影响比较大。比如某个数据历史上访问次数很高，但某个时间点后不再被访问，却迟迟不能被淘汰。

#### 1.3 LRU

最近最少使用，相对于仅考虑时间的FIFO和仅考虑访问频率的LFU，LRU相对平衡。LRU认为，如果数据最近被访问，那么将来被访问的概率也会更高。实现上用一个队列模拟时间序列，如果某条记录被访问，则移动到队尾，那么队首就是最近最少访问的数据。

### 2. LRU算法实现

#### 2.1 核心数据结构

核心数据结构如下图：

![IMG_3374](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/IMG_3374.JPG)

- 为什么使用双链表而不使用单链表？

双链表有前驱和后继节点，删除的时间复杂度为O(1)。

- 为什么双链表中还需要存储key，而不只存储value?

删除最近最少使用节点的时候，需要通过节点获取对应的key，然后再删除哈希表中的键值对。

#### 2.2 核心方法

查找方法`Get()`时间复杂度为O(1),核心代码如下：

```go
// Get look up a key's value
func (c *Cache) Get(key string) (value Value, ok bool) {
	if ele, ok := c.cache[key]; ok {
		c.ll.MoveToFront(ele)
		kv := ele.Value.(*entry)
		return kv.value, true
	}
	return
}
```

插入方法`Add()`时间复杂度为O(1)，流程图如下：

![IMG_3376](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/IMG_3376.JPG)

核心代码如下：

```go
// Add adds a value to cache
func (c *Cache) Add(key string, value Value) {
	if ele, ok := c.cache[key]; ok {
		// 如果存在
		c.ll.MoveToFront(ele)
		kv := ele.Value.(*entry)
		c.nBytes += int64(value.Len()) - int64(kv.value.Len())
		kv.value = value
	} else {
		// 如果不存在
		ele := c.ll.PushFront(&entry{key, value})
		c.cache[key] = ele
		c.nBytes += int64(len(key)) + int64(value.Len())
	}
	// 超过缓存
	// 设置为0表示不对内存大小做限制
	for c.maxBytes != 0 && c.maxBytes < c.nBytes {
		c.RemoveOldest()
	}
}
```

> 注意：缓存大小为0，表示不对内存大小做限制
>
> 使用for循环而不用if，是因为可能插入大对象则需要移除多个缓存记录。

## 单机并发缓存

### 1. sync.Mutex

Sync.Mutex是一个互斥锁，可以由不同协程加锁和解锁。`sync.Mutex` 是 Go 语言标准库提供的一个互斥锁，当一个协程(goroutine)获得了这个锁的拥有权后，其它请求锁的协程(goroutine) 就会阻塞在 `Lock()` 方法的调用上，直到调用 `Unlock()` 锁被释放。

### 2. 支持并发读写

用`sync.Mutex`封装LRU的方法，使之支持并发读写。

先抽象一个**只读的数据结构**`ByteView`用来表示缓存值。ByteView 只有一个数据成员，`b []byte`，b 将会存储真实的缓存值，并且`b`是只读的，返回的是`b`的一个拷贝，防止缓存值被外部程序修改。深拷贝核心代码如下：

```go
func cloneBytes(b []byte) []byte {
	c := make([]byte, len(b))
	copy(c, b)
	return c
}
```

为lru添加并发读写，`cache.go`实例化了lru,并封装了`add()、get()`方法。并且封装的方法是私有的，不会被其他包使用。代码如下：

```go
type cache struct {
	mu sync.Mutex
	lru *lru.Cache
	cacheBytes int64
}

func (c *cache) add(key string, value ByteView) {
	c.mu.Lock()
	defer c.mu.Unlock()

	if c.lru == nil {
		c.lru = lru.New(c.cacheBytes, nil)
	}
	c.lru.Add(key, value)
}

func (c *cache) get(key string) (value ByteView, ok bool) {
	c.mu.Lock()
	defer c.mu.Unlock()

	if c.lru == nil {
		return
	}

	if v, ok := c.lru.Get(key); ok {
		return v.(ByteView), ok
	}
	return
}
```

> 注意：在 `add` 方法中，判断了 `c.lru` 是否为 nil，如果等于 nil 再创建实例。这种方法称之为**延迟初始化(Lazy Initialization)**，一个对象的延迟初始化意味着该对象的创建将会延迟至第一次使用该对象时。主要用于提高性能，并减少程序内存要求。

### 3. 主结构体Group

Group是核心数据结构，负责与用户交互并且控制缓存值存储与获取流程。

![image-20211208101552039](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211208101552039.png)

#### 3.0 接口技巧

- 让编译器检查，确保类型实现了指定的接口

`var _ 指定接口 = (类型)(零值)`。比如代码中有如下一段：

```go
var _ PeerGetter = (*httpGetter)(nil)
```

就是为了判断`*httpGetter`是否实现了`PeerGetter`接口。

- 接口型函数

如果某个函数其中一个参数是该接口类型，**那么既可以将普通的函数类型（函数签名必须相同，需要强制转换）作为参数传入，也可以将实现了该接口的结构体（实例化的结构体变量）作为参数传入**。使用更加灵活。

> 注意：
>
> 1. 接口型函数的接口类型只能有一个方法，这样才能让某一个函树类型实现该接口。
> 2. 定义一个函数类型 F，并且实现接口 A 的方法，然后在这个方法中调用自己。这是 Go 语言中将其他函数（参数返回值定义与 F 一致）转换为接口 A 的常用技巧。

#### 3.1 回调Getter

分布式缓存并没有实现支持多种数据源的配置，**原因有两点：一是数据源种类太多，没办法一一实现，二是扩展型不好**。因此设计了一个回调函数(callback)，当缓存不存在时，调用这个函数，得到源数据。回调函数的设计采用了接口型函数技巧，代码如下：

```go
type Getter interface {
	Get(key string) ([]byte, error)
}

type GetterFunc func(key string) ([]byte, error)

func (f GetterFunc) Get(key string) ([]byte, error) {
	return f(key)
}
```

#### 3.2 Group的定义

```go
// A Group is a cache namespace and associated data loaded spread over
type Group struct {
	name      string
	getter    Getter
	mainCache cache
}

var (
	mu     sync.RWMutex
	groups = make(map[string]*Group)
)
```

一个Group可以认为是一个缓存的命名空间，每个Group拥有一个唯一的名称`name`。第二个属性`getter Getter`，即缓存未命中时获取源数据的回调。第三个属性`mainCache cache`，即并发缓存。

每个Goup存储在全局变量`groups`中。

#### 3.3 Group的Get方法

这时整个分布式缓存最为核心的方法，实现了上述流程的(1)和(3)过程。代码如下：

```go
// Get value for a key from cache
func (g *Group) Get(key string) (ByteView, error) {
	if key == "" {
		return ByteView{}, fmt.Errorf("key is required")
	}

	if v, ok := g.mainCache.get(key); ok {
		log.Println("[GoCache] hit")
		return v, nil
	}
  // 缓存未命中，上述流程(3),回调函数，返回缓存值并将缓存值添加到对应缓存中
	return g.load(key)
}
```

## HTTP服务端

分布式缓存需要实现节点间通信，本项目使用基于HTTP的通信机制，如果一个节点启动了HTTP服务，那么这个节点就可以被其他节点访问。

首先创建一个结构体`HTTPPool`，作为承载节点间HTTP通信的核心数据结构。如下：

```go
type HTTPPool struct {
	// this peer's base URL, e.g. "https://example.net:8000"
	self     string
	basePath string
}
```

`HTTPPool`有两个参数，一个是`self`用来记录自己的地址，包括主机名/IP和端口；另一个是`basePath`，作为节点间通信地址的前缀，默认为`/_gocache/`即`http://example.com/gocache/`开头的请求，就用于节点间的访问。**因为一个主机还可能承载其他的服务，加一段Path是一个好习惯**。

`ServeHTTp`方法代码如下：

```go
// ServeHTTP handle all http requests
func (p *HTTPPool) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if !strings.HasPrefix(r.URL.Path, p.basePath) {
		panic("HTTPPool serving unexpected path: " + r.URL.Path)
	}
	p.Log("%s %s", r.Method, r.URL.Path)
	// /<basepath>/<groupname>/<key> required
	parts := strings.SplitN(r.URL.Path[len(p.basePath):], "/", 2)
	if len(parts) != 2 {
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}

	groupName := parts[0]
	key := parts[1]

	group := GetGroup(groupName)
	if group == nil {
		http.Error(w, "no such group: "+groupName, http.StatusNotFound)
		return
	}

	view, err := group.Get(key)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/octet-stream")
	w.Write(view.ByteSlice())
}
```

首先判断访问路径的前缀是否是`basePath`，**约定访问路径格式为`/<basepath>/<groupname>/<key>`**，通过groupname得到group实例，再使用`group.Get(key)`获取缓存值。最后通过`w.Write()`将缓存值最为httpResponse的body返回。

## 一致性哈希

### 1. 为什么使用一致性哈希

一致性哈希算法是本项目从单节点走向分布式节点的一个重要环节。

#### 1.1 该访问谁？

对于分布式缓存来说，当一个节点接收到请求，如果该节点并没有存储缓存值，将面临从谁哪儿获取数据。

假设包括自己在内有10个节点。假如第一次随机选取了节点1，节点1从数据源获取到数据的同时缓存该数据；那第二次只有1/10的概率再次选择节点1，9/10的概率选择其他9个节点，如果选择了其他节点，就意味着需要再一次从数据源获取数据。这样会导致：**一是缓存效率低，而是各个节点都存储着相同的数据，浪费了大量存储空间**。

使用哈希算法可以解决上述问题，即对于给定的key，每次都选择同一个节点。

![image-20211208144833102](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211208144833102.png)

#### 1.2 节点数量变化了怎么办？

虽然取Hash值解决了缓存性能问题，但没有考虑节点数量变化的场景。分布式中，某一个节点出问题的概率很大，假设某一个节点出问题，那么之前 `hash(key) % 10` 变成了 `hash(key) % 9`，也就意味着几乎缓存值对应的节点都发生了改变。容易引起**缓存雪崩**。

> 缓存雪崩：缓存在同一时刻全部失效，造成瞬间DB请求量大，压力骤增，引起雪崩。常因为缓存服务器宕机或缓存设置了相同的过期时间引起。

**一致性哈希可以解决缓存雪崩问题。**

### 2. 算法原理

#### 2.1 步骤

一致性哈希算法将 key 映射到 2^32 的空间中，将这个数字首尾相连，形成一个环。在新增/删除节点时，只需要重新定位该节点附近的一小部分数据，而不需要重新定位所有的节点。

- 计算节点/机器(通常使用节点的名称、编号和 IP 地址)的哈希值，放置在环上。
- 计算key的哈希值，放置在换上，顺时针寻找到的第一个节点，就是应该选取的节点/机器。

![image-20211208145751780](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211208145751780.png)

环上有 peer2，peer4，peer6 三个节点，`key11`，`key2`，`key27` 均映射到peer2，`key23` 映射到 peer4。此时，如果新增节点/机器 peer8，假设它新增位置如图所示，那么只有 `key27` 从 peer2 调整到 peer8，其余的映射均没有发生改变。

#### 2.2 数据倾斜问题

如果服务器的节点过少，容易引起 key 的倾斜。例如上面例子中的 peer2，peer4，peer6 分布在环的上半部分，下半部分是空的。那么映射到环下半部分的 key 都会被分配给 peer2，key 过度向 peer2 倾斜，**缓存节点间负载不均**。为了解决这个问题，**引入了虚拟节点概念，一个真实节点对应多个虚拟节点。**

假设 1 个真实节点对应 3 个虚拟节点，那么 peer1 对应的虚拟节点是 peer1-1、 peer1-2、 peer1-3（通常以添加编号的方式实现），其余节点也以相同的方式操作。

- 计算虚拟节点的哈希值，放置在环上。
- 计算 key 的 Hash 值，在环上顺时针寻找到应选取的虚拟节点，例如是 peer2-1，那么就对应真实节点 peer2。

**虚拟节点扩充了节点的数量，解决了节点较少的情况下数据容易倾斜的问题。而且代价非常小，只需要增加一个字典(map)维护真实节点与虚拟节点的映射关系即可。**

### 3. 实现

相关结构体定义如下：

```go
// Hash maps bytes to uint32
type Hash func(data []byte) uint32

type Map struct {
  // 默认为crc32.ChecksumIEEE算法，也可以自定义。
	hash Hash
	// 虚拟节点倍数
	replicas int
	// 哈希环
	keys []int  // sorted
	// 虚拟节点与真实节点的隐射，键是虚拟节点哈希值，值是真实节点名称
	hashMap map[int]string
}
```

添加真实节点`Add()`方法：

```go
// Add adds some keys to the hash
func (m *Map) Add(keys ...string) {
	for _, key := range keys {
		for i := 0; i < m.replicas; i++ {
			// 虚拟节点名称是strconv.Itoa(i) + key
			hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
			m.keys = append(m.keys, hash)
			m.hashMap[hash] = key
		}
	}
	sort.Ints(m.keys)
}
```

> 注意：真实节点并没有计算hash值添加到哈希环上。

节点选择`Get()`方法:

```go
// Get gets the closest item in the hash to the provided key
func (m *Map) Get(key string) string {
	if len(m.keys) == 0 {
		return ""
	}
	// 计算key对应的哈希值
	hash := int(m.hash([]byte(key)))
	// 二分查找对应虚拟节点
	idx := sort.Search(len(m.keys), func(i int) bool {
		return m.keys[i] >= hash
	})
	// 如果idx==len(m.keys),说明应该选择m.keys[0]
	return m.hashMap[m.keys[idx%len(m.keys)]]
}
```

## 分布式节点

上述流程图(2)，从远程节点获取缓存值，细化成如下：

![image-20211208151734844](https://narcissusblog-img.oss-cn-beijing.aliyuncs.com/uPic/file-12/image-20211208151734844.png)

### 1. 抽象PeerPicker

抽象出两个接口

- 接口PeerPicker的`PickPeer()`方法用于根据传入的key选择相应节点PeerGetter
- 接口PeerGetter的`Get()`方法用于从对应group查找缓存值。

### 2. 节点选择与HTTP客户端实现

第一步，创建具体的HTTP客户端类`httpGetter`，实现PeerGetter接口。

- baseURL表示将要访问的远程节点的地址，例如`http://exmple.com/gocache/`。

代码如下：

```go
type httpGetter struct {
	baseURL string
}

// Get http客户端方法，访问远程服务端节点从对应group查找key的缓存值
func (h *httpGetter) Get(group string, key string) ([]byte, error) {
	u := fmt.Sprintf(
		"%v%v/%v",
		h.baseURL,
		url.QueryEscape(group),
		url.QueryEscape(key),
	)
	// 发送HTTP请求
	res, err := http.Get(u)
	if err != nil {
		return nil, err
	}
	defer res.Body.Close()

	if res.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("server returned: %v", res.Status)
	}

	bytes, err := ioutil.ReadAll(res.Body)
	if err != nil {
		return nil, fmt.Errorf("reading response body: %v", err)
	}

	return bytes, nil
}
```

第二步，为HTTPPool添加节点选择功能。

HTTPPool结构体增加成员变量：

- `peers`，类型是一致性哈希算法的`Map`，用来根据具体的key选择节点。
- `httpGetters`，映射远程节点与对应的httpGetter。每一个远程节点对应一个httpGetter。

第三步，实现PeerPicker接口，代码如下：

```go
// Set 实例化一致性哈希算法，并添加传入的节点并为每个节点创建一个HTTP客户端httpGetter
func (p *HTTPPool) Set(peers ...string) {
	p.mu.Lock()
	defer p.mu.Unlock()

	p.peers = consistenthash.New(defaultReplicas, nil)
	p.peers.Add(peers...)
	p.httpGetters = make(map[string]*httpGetter, len(peers))
	for _, peer := range peers {
		p.httpGetters[peer] = &httpGetter{baseURL: peer + p.basePath}
	}
}

// PickPeer 根据key返回节点,即节点对应的HTTP客户端
func (p *HTTPPool) PickPeer(key string) (PeerGetter, bool) {
	p.mu.Lock()
	defer p.mu.Unlock()

  // 如果peer不为空并且不是自身节点
	if peer := p.peers.Get(key); peer != "" && peer != p.self {
		p.Log("Pick peer %s", peer)
		return p.httpGetters[peer], true
	}
	return nil, false
}
```

则**HTTPPool既具备提供HTTP服务的能力，也具备根据具体的key，创建HTTP客户端从远程节点获取缓存值的能力**。

### 3. 实现主流程

- `RegisterPeers()`方法，将实现了PeerPicker接口的HTTPPool注入到Group中。
- `getFromPeer()`方法，访问远程节点peerGetter，获取缓存值。

```go
// RegisterPeers 将实现了PeerPicker接口的HTTPPool注入到Group中
func (g *Group) RegisterPeers(peers PeerPicker) {
	if g.peers != nil {
		panic("RegisterPeerPicker called more than once")
	}
	g.peers = peers
}

// getFromPeer 访问远程节点peerGetter,获取key缓存值
func (g *Group) getFromPeer(peer PeerGetter, key string) (ByteView, error) {
	bytes, err := peer.Get(g.name, key)
	if err != nil {
		return ByteView{}, err
	}
	return ByteView{b: bytes}, nil
}
```

## 防止缓存击穿

### 1. 缓存雪崩、缓存击穿与缓存穿透

> **缓存雪崩**：缓存在同一时刻全部失效，造成瞬时DB请求量大、压力骤增，引起雪崩。缓存雪崩通常因为缓存服务器宕机、缓存的key设置了相同的过期时间等引起。
>
> **缓存击穿**：一个存在的key，在缓存过期的一刻，同时有大量的请求，这些请求都会击穿到DB，造成瞬时DB请求量大，压力骤增。
>
> **缓存穿透**：查询一个不存在的数据，因为不存在则不会写到缓存中，所以每次都会去请求DB，如果瞬间流量过大，穿透到DB，导致宕机。

### 2. singleflight的实现

如果并发了N个相同的请求。假设对数据库的访问没有做任何限制，很有可能向数据库也发起N次请求，容易导致缓存击穿或穿透。即使对数据库做了防护，HTTP请求也是非常消耗资源的操作，针对相同的key,也没有必要向远程缓存节点发起N次相同的请求。这种情况下，如何做到只向远端节点发起一次请求呢？

#### 2.1 相关结构体定义

```go
type call struct {
	wg sync.WaitGroup
	val interface{}
	err error
}

type Group struct {
	mu sync.Mutex
	m map[string]*call
}
```

- `call`代表正在进行中或已经结束的请求。使用`sync.WaitGroup`锁。
- `Group`是singleflight是主数据结构，管理不同key的请求（call）。

#### 2.2 核心方法的实现

```go
// Do 无论Do被调用多少次，函数fn只会被调用一次，等待fn调用结束，返回返回值或错误。
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	// 延迟初始化，提高内存使用效率
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()  // 如果请求正在进行，则等待
		return c.val, c.err  // 请求结束，返回结果
	}
	c := new(call)
	c.wg.Add(1)  // 发起请求前加锁
	g.m[key] = c  // 添加到g.m,表明key已经有对应的请求在处理
	g.mu.Unlock()

	c.val, c.err = fn()  // 调用fn，发起请求
	c.wg.Done()  // 请求结束

	g.mu.Lock()
	delete(g.m, key)  // 更新g.m
	g.mu.Unlock()

	return c.val, c.err  // 返回结果
}
```

`g.mu`是保护Group的成员变量`m`不被并发读写而加上的锁。

>并发协程之间不需要消息传递，非常适合`sync.WaitGroup`：
>
>- `wg.Add(1)`锁加1。
>- `wg.Wait()`阻塞，直到锁被释放。
>- `wg.Done()`锁减1。

### 3. singleflight的使用

使用singleflight中的Do方法对服务端的`load()`方法进行封装：

```go
func (g *Group) load(key string) (value ByteView, err error) {
	viewi, err := g.loader.Do(key, func() (interface{}, error) {
		if g.peers != nil {
			if peer, ok := g.peers.PickPeer(key); ok {
				if value, err = g.getFromPeer(peer, key); err == nil {
					return value, nil
				}
				log.Println("[GeeCache] Failed to get from peer", err)
			}
		}

		return g.getLocally(key)
	})

	if err == nil {
		return viewi.(ByteView), nil
	}
	return
}
```

## 使用protobuf通信

### 1. 为什么要使用protobuf

> protobuf即Protocol Buffers，Google开发的一种数据描述语言，是一种轻便高效的结构化数据存储格式，与语言、平台无关，可扩展可序列化。protobuf以二进制方式存储，占用空间小。

protobuf 广泛地应用于远程过程调用(RPC) 的二进制传输，**使用 protobuf 的目的非常简单，为了获得更高的性能。传输前使用 protobuf 编码，接收方再进行解码，可以显著地降低二进制传输的大小。另外一方面，protobuf 可非常适合传输结构化数据，便于通信字段的扩展**。

使用步骤：

- 按照protobuf的语法，在`.proto`文件中定义数据结构，并使用`protoc`生成Go代码。
- 在项目代码中引用生成的Go代码。

### 2. 使用protobuf通信

`.proto`文件中定义数据结构如下：

```protobuf
syntax = "proto3";

package gocachepb;
option go_package = "./";

message Request {
  string group = 1;
  string key = 2;
}

message Response {
  bytes value = 1;
}

service Groupcache {
  rpc Get(Request) returns (Response);
}
```

`serveHTTP()`使用`proto.Marshal()`编码HTTP响应，`Get()`中使用`proto.Unmarshal()`解码HTTP响应。

```go
func (p *HTTPPool) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // ...
	// Write the value to the response body as a proto message.
	body, err := proto.Marshal(&pb.Response{Value: view.ByteSlice()})
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/octet-stream")
	w.Write(body)
}

func (h *httpGetter) Get(in *pb.Request, out *pb.Response) error {
	u := fmt.Sprintf(
		"%v%v/%v",
		h.baseURL,
		url.QueryEscape(in.GetGroup()),
		url.QueryEscape(in.GetKey()),
	)
    res, err := http.Get(u)
	// ...
	if err = proto.Unmarshal(bytes, out); err != nil {
		return fmt.Errorf("decoding response body: %v", err)
	}

	return nil
}
```

