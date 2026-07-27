+++
title = "写一个 ISO 8583 的 SDK"
description = "从上千页日文规格书到一个 Go SDK: 反射构造报文, JIS8 编码, 全双工连接池, 会话与密钥, 以及一个叫李鬼的假中心"
date = "2022-12-31T17:40:11+09:00"
draft = false
[taxonomies]
tags = ["Golang", "Computer Network", "Payment"]
+++

业务上要和日本某卡组织的交换中心直连, 走 ISO 8583.

最初分配给我的任务是写一个测试挡板, 让业务那边先能跑起来, 不用等到真的接上对端. 但一动手就发现顺序反了 —— 挡板要能收下我方发出去的报文, 就得会解包; 要能回一条对端格式的响应, 就得会封包; 要像对端那样建连接, 开局, 换密钥, 那这些也一个都少不了. 换句话说, **不先把这套东西做完, 连挡板都写不出来**.

市面上的 Go 库也不太符合需求: 要么只做报文的封装解包, 要么假设你跑在一个标准的 request-response 模型上, 而对端的规格书里写的是全双工, 双向主动发起, 定期换密钥.

所以最后做出来的不是一个报文库, 也不只是一个挡板. 封包解包只是最底下的一层, 上面还压着连接, 会话, 密钥, 以及一整套给业务用的接口 —— 是一个 SDK. 而当初要的那个挡板, 反倒成了它的一个使用者.

趁年末还记得, 把几个设计决定记一下. 具体的接续规格是有 NDA 的, 所以下面的代码都是脱敏和简化过的, 只留设计本身.

## ISO 8583 是什么

简单说, 是刷卡交易的报文标准. 你在便利店刷一张卡, 从 POS 机到收单机构到卡组织到发卡行, 这一路上跑的多半就是 ISO 8583 或者它的某个方言.

报文的结构非常有年代感:

```text
+--------+------------------+---------------------------+
|  MTI   |      Bitmap      |          Fields           |
| 4 bytes|  8 or 16 bytes   |         变长               |
+--------+------------------+---------------------------+
```

- **MTI** (Message Type Indicator), 4 位数字, 说明这条报文是什么, 比如授权请求, 冲正, 网络管理.
- **Bitmap**, 64 个 bit, 第 N 个 bit 为 1 表示第 N 个域出现在报文里. 第 1 个 bit 比较特殊, 它为 1 表示后面还跟了第二个 bitmap, 即这条报文用到了 65~128 域.
- **Fields**, 按域号从小到大依次排列. 每个域是定长还是变长, 是数字还是字符, 用什么编码, 都由规格书规定.

它是个 1980 年代的标准, 所以处处是省字节的痕迹: 域出现与否用 bit 表示而不是用 key, 数字用 BCD 压缩成半个字节, 变长域的长度头也用 BCD. 在今天看这些设计很别扭, 但在专线按字节计费的年代是合理的.

不过有一件事要先分清楚: **ISO 8583 是公开的标准, 但它管的基本上只有报文**.

它规定了 MTI, bitmap, 每个域号大致对应什么含义, 以及各类域的类型和长度表示法. 至于连接怎么建, 会话怎么开怎么关, 心跳多久发一次, 密钥怎么下发怎么轮换, 出了错回什么码, 日切怎么对账 —— 一概不管. 这些全部由使用这张网的双方自己约定.

而且就算只看报文这一层, 各方也都有自己的变体. 同一个域号, 这张网拿来放这个, 那张网拿来放那个; 有的域整张网都不用; 长度头的编码, 字符集, 定长域往哪边补位, 也各家不同.

所以"支持 ISO 8583"这句话本身几乎不提供什么信息, 真正要问的是"支持哪一张网的 ISO 8583". 而这个问题的答案, 只写在接续规格书里.

## 最大的困难是文档

必须先说这个, 因为整个项目里最难的部分从来不是代码.

规格书是几百页的日文 PDF, 还远不止一份. 印象里前后拿到的有十来册: 基本篇和电文篇算主干, 然后**每多一个功能就多一册** —— IC 卡是单独一册, 非接触的タッチ又是一册, PIN 相关的再一册, 加上各种别册和历年的改订通知. 摞起来轻松上千页.

更麻烦的是它们的年龄. 这套东西是上世纪就在跑的, 文档也是那时候留下来一路改过来的, 于是你会读到这样的内容:

- 排版是等宽字体画的表格, 有些页干脆是扫描件, 复制不出文字, 只能对着屏幕一个字一个字敲.
- 术语是昭和的. 電文, 仕向, 宛先, 加盟店, 開局, 閉局, 締め, カットオーバー —— 每一个都要先搞清楚它在这个语境下具体指什么, 而不是字面意思.
- 片假名外来语和汉字词混着用. レコード, フィールド, ヘッダ 是一套, 電文長, 可変長 是另一套, 指的却是同一层东西.
- **甚至还有 X.25 的描述.** 分组交换网, 上世纪七十年代的东西. 早就跑在 TCP/IP 上了, 那一段却还留着. 读的时候要一边分辨哪些已经不适用, 一边小心那些看着过时, 实际上对端还在严格执行的部分.

但真正累人的不是这些, 而是**这些文档之间没有链接**.

倒不是说它们写得有矛盾 —— 恰恰相反, 内容相当严谨, 该规定的都规定了. 问题是同一件事的信息被按功能切开, 摊在好几册里.

举个实际的例子: 要搞清楚一笔带 PIN 的 IC 卡交易怎么发, 域的位置和长度在电文篇, PIN block 怎么算, 用哪把密钥加密在 PIN 那一册, IC 卡数据域里那串 TLV 的构成在 IC 卡那一册, 而这三册对同一条报文各自只讲自己负责的那一部分. 谁也没错, 但没有一册能单独回答"这条报文到底长什么样".

**你就是那个超链接.** 没有交叉引用, 没有索引, 扫描的那几页连全文检索都做不了, 全靠自己记住哪一条写在哪一册的第几页. 所以前期有相当一部分时间, 其实是在给自己建索引: 术语的对照, 每个域的出处, 各册之间的对应关系. 这件事没有任何技术含量, 但跳过它的代价是后面每查一次都要重新翻一遍.

实在读不出来的地方也有, 只能记下来攒着, 攒够一批再去问对端 —— 问一轮的周期以周计, 所以清单要攒得够厚才划算.

所以真正的工作量分布大概是: 读文档, 整理术语, 建索引占了一大半, 写代码反而是最快的部分. 这也直接决定了后面几乎所有的设计.

因为读文档这件事最大的风险不是读慢了, 而是**读错了却不知道**. 一个人对着上千页 PDF 读出来的理解, 如果只留在他脑子里, 或者只写成一页 wiki, 那它既不会被 review, 也不会被验证, 出了问题还查不到源头. 所以从一开始就定了一个原则:

> 凡是规格书里写的约束, 都要变成代码里能跑的东西.

域的长度, 编码方式, 是定长还是变长, 在哪种电文里必须出现, 取值范围有哪些 —— 全部写进结构体和校验器, 而不是写进注释或者文档. 这样"我对规格书的理解"就成了一个可以被 review, 被测试, 被后来人修正的东西.

而一旦这么想, 事情就变得很顺: 既然报文无非是"哪些域出现了, 每个域怎么编码", 那它完全可以用 Go 的结构体和 tag 描述出来.

## 用 struct tag 描述报文

底层的封包解包部分, 我们没有从零写, 而是基于 [ideazxy/iso8583](https://github.com/ideazxy/iso8583) 的思路做的扩展. 它的核心抽象是一个接口:

```go
// Iso8583Type interface for ISO 8583 fields
type Iso8583Type interface {
	// Bytes 把当前域编码成字节
	Bytes(encoder, lenEncoder, length int) ([]byte, error)

	// Load 从字节流解出当前域, 返回实际消费的字节数
	Load(raw []byte, encoder, lenEncoder, length int) (int, error)

	// IsEmpty 判断域是否为空
	IsEmpty() bool
}
```

然后 `Numeric`, `Alphanumeric`, `Binary`, `Llvar`, `Lllvar`, `Llnumeric`, `Lllnumeric` 各自实现它. `Ll` 和 `Lll` 前缀是 ISO 8583 的黑话, 指长度头是 2 位还是 3 位十进制数的变长域.

于是一条报文就是一个结构体:

```go
type Body struct {
	F2  *iso8583.Llvar        `field:"2"  length:"19" encode:"jis8"` // 会员号
	F3  *iso8583.Numeric      `field:"3"  length:"6"  encode:"jis8"` // 处理码
	F4  *iso8583.Numeric      `field:"4"  length:"12" encode:"jis8"` // 交易金额
	F11 *iso8583.Numeric      `field:"11" length:"6"  encode:"jis8"` // 系统跟踪号
	F12 *iso8583.Numeric      `field:"12" length:"12" encode:"jis8"` // 交易时间
	F37 *iso8583.Alphanumeric `field:"37" length:"12" encode:"jis8"` // 检索参考号
	F52 *iso8583.Binary       `field:"52" length:"8"`                // PIN block
	F55 *iso8583.LllBinary    `field:"55" length:"255"`              // IC 卡数据
}
```

`Bytes()` 的时候, 用反射遍历这个结构体, 从 tag 里读出域号, 长度和编码, 域号决定它在 bitmap 里点哪个 bit, 编码决定它怎么变成字节:

```go
func parseFields(msg interface{}) map[int]*fieldInfo {
	fields := make(map[int]*fieldInfo)

	v := reflect.Indirect(reflect.ValueOf(msg))
	for i := 0; i < v.NumField(); i++ {
		// 指针为 nil 的域, 直接跳过, 不进 bitmap
		if isPtrOrInterface(v.Field(i).Kind()) && v.Field(i).IsNil() {
			continue
		}
		sf := v.Type().Field(i)
		index, _ := strconv.Atoi(sf.Tag.Get(TAG_FIELD))
		// ... 解析 encode / length tag
		field, ok := v.Field(i).Interface().(Iso8583Type)
		if !ok {
			panic("field must be Iso8583Type")
		}
		fields[index] = &fieldInfo{index, encode, lenEncode, length, field}
	}
	return fields
}
```

这里有两个我觉得挺舒服的地方.

第一, **所有域都用指针**. Go 没有 optional, 而 ISO 8583 里"域不存在"和"域是空字符串"是两件不同的事 (前者 bitmap 上是 0, 后者是 1 加一个长度为 0 的变长域). 用指针 nil 表达"不存在", 用 `IsEmpty()` 表达"存在但为空", 语义就对上了. 调用方只需要 `models.WithF39("000")` 这样填自己关心的域, 剩下的自然不出现在报文里.

第二, **bitmap 不需要人来维护**. 老一点的实现里, 常常是业务代码自己去 set bit, 然后再 append 数据, 一旦漏了一处就是一条对端解不开的报文. 交给反射之后, 结构体里填了什么, 报文里就有什么.

解包的方向对称: 注册 MTI 与模板结构体的映射, 收到报文先读 MTI, 找到对应的 `reflect.Type`, `reflect.New` 一个出来, 再按 bitmap 逐位 `Load`:

```go
func (p *Parser) Register(mti string, tpl interface{}) (err error) {
	v := reflect.ValueOf(tpl)
	p.messages[mti] = reflect.Indirect(v).Type()
	return nil
}
```

有一个必须处理的细节: `reflect.New` 出来的结构体, 里面所有指针域都是 nil, 而 `Load` 需要往里写. 所以要先把指针域挨个 `reflect.New` 初始化一遍, 解完之后再把没被 bitmap 点到的域还原成 nil —— 否则 "这条报文里有没有 F39" 这个判断就失效了.

反射代码的另一个代价是, 大量的错误没法在编译期发现, tag 写错, 类型写错, 都要到运行时才 panic. 这个库的做法是在每个对外的入口上挂一个 recover, 把 panic 翻译成 error:

```go
func (m *Message) Bytes() (ret []byte, err error) {
	defer func() {
		if r := recover(); r != nil {
			err = errors.New("Critical error:" + fmt.Sprint(r))
			ret = nil
		}
	}()
	// ...
}
```

这是个务实但不优雅的选择. 一条畸形报文不该把整个网关搞挂, 但用 recover 兜底也意味着错误信息很难看. 更好的做法应该是在注册模板的时候就把结构体校验一遍, 把错误提前到启动期.

## 两种编码: BCD 和 JIS8

### BCD

BCD 是用 4 个 bit 表示 1 位十进制数, 于是 1 个字节能放 2 位数字, `"1234"` 编码后就是 `\x12\x34`. 对于金额, 日期, 长度头这些纯数字字段, 直接省一半.

麻烦在奇数长度. `"119"` 有 3 位, 占 1.5 字节, 那多出来的半个字节补在哪一边? 规格书里两种都有, 所以要两个函数:

```go
// 左对齐: "119" -> "1190" -> \x11\x90
func lbcd(data []byte) []byte {
	if len(data)%2 != 0 {
		return bcd(append(data, "0"...))
	}
	return bcd(data)
}

// 右对齐: "119" -> "0119" -> \x01\x19
func rbcd(data []byte) []byte {
	if len(data)%2 != 0 {
		return bcd(append([]byte("0"), data...))
	}
	return bcd(data)
}
```

解码时同样要知道原始的十进制长度才能把补的那个 0 切掉, 所以 `Load` 的签名里一直带着 `length`. 这也是为什么 tag 里的 `length` 对于 BCD 域指的是**十进制位数**而不是字节数 —— 一个很容易踩的坑, 尤其是 `Binary` 类型的 `length` 指的又是字节数.

### JIS8

这是这个项目里最日本的部分.

Go 的 `string` 是 UTF-8, 而线路上跑的是 Shift-JIS. 具体一点, 报文里的日文用的是 JIS X 0201 的半角片假名 (单字节, 0xA1~0xDF 区间) 加 JIS X 0208 的双字节汉字假名. `ｱｲｳｴｵ` 这种半角片假名今天只在 ATM 和收据上还能见到, 但在这类报文里是主力.

转换本身不难, `golang.org/x/text/encoding/japanese` 直接给:

```go
func ToShiftJIS(str string) (res []byte, err error) {
	transformed, err := io.ReadAll(
		transform.NewReader(strings.NewReader(str), japanese.ShiftJIS.NewEncoder()),
	)
	// ...
}
```

难的是, **转换应该发生在什么时候**.

一开始我们把域直接存成 `[]byte`, 让调用方自己转. 结果是业务代码里到处是 `ToShiftJIS`, 而且一旦漏了一处, 报文能发出去, 对端也能收下, 只是商户名变成乱码 —— 这种 bug 联调的时候非常难查.

后来改成变长域同时持有两种表示:

```go
type Llvar struct {
	ByteValue []byte
	Value     string // UTF-8 字符串, 如果填了此值, 会以此值为准做转换
	Valid     bool
}
```

业务代码只碰 `Value`, 编码的事情推迟到 `Bytes()` 里, 按 tag 上的 `encode` 决定是原样输出还是转 Shift-JIS; 解包时反过来, `Load` 之后 `Value` 里拿到的一定是可以直接打日志, 直接进数据库的 UTF-8. 编码从此只在库的边界上发生一次.

顺带一个必须记住的点: **变长域的长度头算的是编码后的字节数, 不是字符数**. `"東京"` 在 UTF-8 里是 6 字节, 在 Shift-JIS 里是 4 字节, 长度头要写 4. 同理, 定长域补空格补到的也是字节数. 所以 `len(l.ByteValue)` 这个写法必须在转换之后, 顺序反了就是一条长度对不上的废报文.

测试数据也是个问题. 为了压变长域的边界, 需要随机生成合法的 Shift-JIS 字符串, 于是写了个工具, 把 ASCII 的符号区间和 0xA1~0xDF 的半角片假名区间拼起来, 解码成 rune 再随机取:

```go
for i := uint8('\x21'); i <= uint8('\x2f'); i++ {
	b = append(b, i)
}
// ... 其他 ASCII 符号区间
for i := uint8('\xa1'); i <= uint8('\xdf'); i++ {
	b = append(b, i)
}
```

代码里还留着一行 `// TODO handle \xa0, it's very strange` —— 0xA0 在 Shift-JIS 里是未定义的, 但某些实现会把它当空格处理. 规格书里对这个位置也语焉不详, 这个 TODO 估计会留很久.

## TCP 连接

封包解包做完, 剩下的一半工作量在连接上. 这部分基本是照着 [go-redis](https://github.com/redis/go-redis) 抄的, 然后被业务改得面目全非.

抄的部分是 `Conn` 这层封装: 拿一个 `*net.TCPConn`, 套上 `bufio.Reader` / `bufio.Writer`, 配上读写超时和 keepalive.

```go
func Connect(address string) (conn *net.TCPConn, err error) {
	raddr, err := net.ResolveTCPAddr("tcp", address)
	// ...
	conn, err = net.DialTCP("tcp", nil, raddr)
	// ...
	err = conn.SetKeepAlivePeriod(15 * time.Second)
	err = conn.SetKeepAlive(true)
	return
}
```

TCP 是字节流, 没有报文边界, 所以读的时候必须自己划. 好在这类协议都有定长包头, 包头里有全长:

```go
// 先读定长包头
headerBytes := make([]byte, models.HeaderByteLength)
_, err = io.ReadFull(conn.br, headerBytes)
// 从包头里取出全长, 再读剩下的
totalLength, err := message.GetTotalLengthFromHeader()
bodyBytes := make([]byte, totalLength-models.HeaderByteLength)
_, err = io.ReadFull(conn.br, bodyBytes)
```

`io.ReadFull` 在这里是关键. 直接 `Read` 是不保证读满的, 大报文分片到达的时候, 半包就是这么来的.

改得面目全非的部分, 是**这个协议是全双工的**.

Redis 的连接模型是: 从池里取一条连接, 写请求, 阻塞读响应, 还回池里. 一条连接同一时刻只服务一个请求, 请求和响应严格配对.

而卡网这边不是. 对端会主动推送过来通知类电文, 也会在你发出去的请求还没回来的时候先发别的东西过来. 更麻烦的是, 双方是对等的: 我方既要主动连过去, 也要监听端口等对方连过来, 两个方向上都能收发业务电文.

所以连接建立之后, 立刻起一个 goroutine 一直读, 读到的电文全部投进一个 channel:

```go
func (c *Conn) ReadChan(ch chan<- *models.Message) {
	go func() {
		for {
			message, err := connReadMessage(c)
			if err != nil {
				logz.Warnf("read error, %v", err)
				c.Close()
				return
			}
			ch <- message
		}
	}()
}
```

上层拿到的就是一个"所有连接上收到的所有电文"的流, 主动连出去的和被动接进来的连接共用同一个出口:

```go
func (c *Center) ReceiveMessagesInChannel(ch chan<- *models.Message) {
	serverReceiveChan := c.server.ReceiveChannel()
	clientReceiveChan := c.client.ReceiveChannel()
	go func() {
		for {
			select {
			case message = <-serverReceiveChan:
				go loadMessage(ch, message, c.state)
			case message = <-clientReceiveChan:
				go loadMessage(ch, message, c.state)
			}
		}
	}()
}
```

请求和响应的配对, 就交给上层用报文里的流水号去做了 —— 系统跟踪号加交易时间加机构号组成的那个唯一标识. 这件事本来就该在业务层做, 因为超时重发, 冲正, 对账都要用到同一个标识.

## 一点也不云原生

在讲连接池之前, 得先说清楚这张网长什么样, 否则池子的形状会显得莫名其妙.

没有域名, 没有服务发现, 没有负载均衡器. 双方交换的是一份 IP 地址清单, 写死在配置里:

```go
type Config struct {
	Addresses []string `json:"addresses"`
	SourceID  string   `json:"source_id"`
}
```

我方有 m 个节点, 对方给 n 个地址, 于是 **m × n 条连接, 每一条各自建立, 各自保活, 各自重连**. 反方向还有一张: 对方也会主动连过来. 这些连接不是一堆等价的, 可替换的池化资源, 而是一条条有名有姓的线路 —— 断的是哪一条, 就得把哪一条接回来.

所以这套东西和现在习以为常的那一套基本是反着来的:

- **不能随便扩容.** 实例数是和对方约定好的. 加一台机器意味着双方都要改白名单, 走申请流程, 周期以周计. 弹性伸缩在这里不是一个选项.
- **IP 必须是固定的.** 这基本堵死了直接扔进 k8s 让 Pod 飘来飘去的可能, 要么钉在固定节点上, 要么想办法保证出口 IP 稳定.
- **发布是有代价的.** 重启一个实例意味着 n 条连接断开重连, 重连完还要重新开局. 滚动发布要控制节奏, 不能一次全部重启.
- **健康检查没法交给别人.** 没有 LB 帮你把故障节点摘掉, 每条连接是死是活, 只有进程自己知道.

好处是链路上没有中间环节, 出了问题排查路径很短. 代价是建立, 保活, 重连, 选路, 摘除这些活全部落在库里, 没有任何基础设施能替你分担 —— 下面这个连接池, 本质上就是为了把这堆事收在一个地方.

## 连接池

既然读已经不需要占用连接了, 那连接池到底还池什么?

答案是: **只服务于发送**. 池的职责从"借出-独占-归还"退化成了"在一组可用连接里挑一条来写", 也就是选路和负载均衡.

底层的双向链表直接抄了 [redigo](https://github.com/gomodule/redigo), 代码里老老实实留着出处:

```go
// idleList is a double linked list to maintain connections in connPool
// copy from github.com/gomodule/redigo
type idleList struct {
	count       int
	front, back *poolConn
}
```

取连接的时候从链表尾部弹出, 用完从头部塞回去, 天然就是一个轮询:

```go
func (p *ConnPool) GetConnAndPutBack() (conn *Conn, err error) {
	p.mu.Lock()
	defer p.mu.Unlock()
	conn, err = p.getConn()
	if err != nil {
		return
	}
	if !conn.closed {
		// 将取到的连接还回到链表的头
		p.putConn(conn)
	}
	return
}
```

发送的入口带 context 超时, 拿不到可用连接就等, 等到 deadline 再报错. 已经关闭的连接直接丢弃, 不再放回:

```go
func (p *ConnPool) GetActiveConnWithCtx(ctx context.Context) (conn *Conn, err error) {
	for {
		select {
		case <-ctx.Done(): // 已经超时, 直接返回
			return nil, ctx.Err()
		default:
			p.mu.Lock()
			conn, err = p.getConn()
			p.mu.Unlock()
			if err != nil {
				if err == ErrIdleConnNotFound { // 没有空闲连接, 等一下再取
					time.Sleep(time.Millisecond)
					continue
				}
				return
			}
			if conn.closed {
				continue // 关闭的连接无需加回连接池
			}
			return // 返回的 conn 一定是未关闭且可用的
		}
	}
}
```

### 重连

专线也会断, 对端也会重启, 而这类系统的要求是断了要自己接回来, 不能等人来重启进程.

做法是在连接入池的时候, 给它挂一个"守灵"的 goroutine: 阻塞在这条连接的 `closeChan` 上, 一旦连接关闭, 先把活跃计数减掉, 然后按**原来那个**远端地址重新拨号, 拨通了再入池. 注意是原来那个, 而不是从地址清单里重新挑一个 —— 上面说过, m × n 里的每一条都是有名有姓的, 少了哪条就得补哪条.

```go
func (p *ConnPool) AddConn(conn *Conn) (err error) {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.putConn(conn)
	p.increaseActive()
	go func(conn *Conn) {
		<-conn.closeChan
		p.mu.Lock()
		p.decreaseActive()
		p.mu.Unlock()
		if !conn.reconnect {
			return // 被动接入的连接不负责重连
		}
		if conn.parentCloseChan == nil {
			return
		}
		// ... 按 remote addr 重连, 失败退避后重试
	}(conn)
	return
}
```

这里有两个 flag 值得说一下.

`reconnect` 区分这条连接是我方拨出去的还是对方连进来的. 只有前者该重连 —— 对方连进来的连接断了, 该由对方来重连, 我方去连对方的监听端口是完全不同的一件事.

`parentCloseChan` 区分"网络断了"和"应用要关了". 没有这个东西的话, 优雅关停会变成: 你关掉所有连接, 然后每条连接的守灵 goroutine 立刻开始重连, 关不掉. 这个 channel 由上层持有, 关停时先关它, 重连逻辑一看它已经关闭就直接退出.

```go
func IsClosed(ch <-chan interface{}) bool {
	select {
	case <-ch:
		return true
	default:
	}
	return false
}
```

这个模式在 Go 里很常用: 用一个只关不写的 channel 做广播式的取消信号. 现在写的话应该直接用 `context.Context`, 当时是图省事.

## 连接不等于会话

规格书里, 两个中心之间要先发一条开局电文, 收到开局响应之后才允许发业务电文, 收工的时候再发一条闭局.

应用层再握一次手这件事本身一点也不稀奇. Redis 的 `AUTH`, `SELECT`, `HELLO` 就是, MySQL, AMQP, SMTP 也都有自己的一套. TCP 只负责把字节按序送到, 至于"你是谁, 我认不认你, 接下来按什么规矩来", 永远得在应用层谈.

真正值得说的是**粒度**.

Redis 那套是**连接级**的: AUTH 认的是这条连接, SELECT 选的是这条连接的 db, 状态和 socket 同生共死. 连接断了, 状态跟着没; go-redis 重连之后在 `initConn` 里把 HELLO / AUTH / SELECT 重跑一遍, 上层完全无感. 会话和连接是 1:1 的, 所以"会话"这个词在那边根本不需要单独存在.

开局不是. 它是**两个中心之间**的状态, 一次开局覆盖 m × n 所有连接:

- 单条连接断了重连, 不需要重新开局.
- 反过来, 所有连接都好好的, 但会话被闭局了, 一样不能发业务电文.

所以在线状态是存在 `CommonState` 里的, 不是存在 `Conn` 上:

```go
const (
	OnlineStatusClosed OnlineStatus = iota
	OnlineStatusOpening
	OnlineStatusOpened
	OnlineStatusClosing
)
```

发业务电文之前该问的是"会话开着吗", 而不是"连接还在吗". 这两个判断在代码里是分开的, 池子非空只是必要条件.

粒度不同, 后面几件事也就跟着不同了.

**会话要自己判断活性.** TCP 断了你不一定知道 —— 对端主机掉电, 中间的防火墙静默丢掉连接状态, 专线切换, 这些情况下本地的 socket 可能很长时间还停在 ESTABLISHED, 你往里写也不报错. keepalive 探的是链路, 对端进程 hang 死了内核照样回 ACK. 这个问题 Redis 那边由池子处理掉了: 拿到一条坏连接, 丢掉重建, 大不了重试一次, 反正会话跟着连接一起重来. 但在这里连接可以重建, 会话不行, 所以必须有应用层的心跳 (echo test) —— 它超时说明的是"对面那套系统不转了", 而不只是"这条线断了".

**它是对等的.** AUTH 是客户端单方面向服务端证明自己, 服务端不需要反过来 AUTH. 而开局是两个对等中心之间的协商, 哪边都能发起, 哪边都要维护对方的状态. 后面那个假中心之所以要把开局的两个方向都实现一遍, 就是因为这个.

**闭局是要谈的.** TCP 的 FIN 只能表达"我不发了", 表达不了"我准备收工, 麻烦把在途的交易处理完再断", 更不给对方拒绝或者拖一会儿的机会. 而这类系统有营业日, 有日切, 有对账 —— 停机是一件有业务含义的事, 只能在应用层用电文来谈.

最后一个佐证是超时的量级:

```go
var (
	openResponseTimeout     = time.Duration(45 * time.Second)
	closeResponseTimeout    = time.Duration(75 * time.Second)
	echoTestResponseTimeout = time.Duration(90 * time.Second)
)
```

几十秒到一分半, 远大于任何一个 TCP 层面的超时. 这也说明了它们量的到底是什么: 不是网络通不通, 而是对面那套系统还转不转得动.

## 密钥的接口注入

最后一块, 也是我自己最满意的一块.

这类协议的安全部分大致是这样: 业务电文的 body 要加密, 同时要算一个电文认证值挂在包头上防篡改. 用的是 3DES, 链式地把 body 一段段加密, 认证值取最后一段密文的前几个字节 —— 也就是一个 CBC-MAC 的构造:

```go
func CalcAuthenticationCode(data, key []byte) (authCode []byte, err error) {
	// 不足 8 的整数倍, 末尾补 0
	// ...
	input := []byte("\x00\x00\x00\x00\x00\x00\x00\x00")
	for i := 0; i < len(data)/8; i++ {
		current := data[i*8 : (i+1)*8]
		temp, err := Xor(input, current)
		input, err = TripleDesEncrypt(temp, key)
	}
	authCode = input[0:4]
	return
}
```

问题不在算法, 在密钥从哪来.

加密和认证用的是两把不同的工作密钥, 它们会定期轮换: 通过一条专门的密钥交换电文下发, 下发时被主密钥加密, 收到之后解出来才是真正能用的值. 与此同时, 包头里有一个 check digit 字段标识"这条电文用的是哪一把密钥" —— 因为轮换的那一瞬间, 线上会同时存在用新旧两把密钥加密的电文, 你必须能按 check digit 反查出对应的密钥来解.

这就带来一个问题: 密钥的存储和轮换, 完全是业务的事情. 单进程的时候放内存就行, 多进程要放 Redis 或者数据库, 讲究一点的要放 HSM. SDK 不该管这些, 但它又必须在打包和解包的那一刻拿到密钥.

于是把它做成了接口, 由使用方注入:

```go
type SecretHandler interface {
	RetrieveSecret() (secret *models.Secret, err error)
	RetrieveSecretWithCheckDigit(checkDigit []byte) (secret *models.Secret, err error)
	RetrieveSpecificSecret(secret *models.Secret) (err error)
}

type CommonStateHandler interface {
	SecretHandler
	RetrieveOnlineStatus() (status OnlineStatus, err error)
}
```

库里只留调用点. 发送的时候, 网络管理类电文不需要密钥, 业务电文才去取:

```go
func (b *base) SendMessage(message *models.Message) (err error) {
	withoutSecret, err := models.IsMsgTypeCodeNetworkControl(message.MessageType)
	if !withoutSecret {
		message.Secret, err = b.state.RetrieveSecret()
		if err != nil {
			return
		}
	}
	raw, err := message.Bytes()
	// ...
}
```

接收的时候, 先解包头, 从 check digit 反查密钥, 再拿这把密钥去解 body 和验认证值:

```go
func loadMessageBody(message *models.Message, sh SecretHandler) (err error) {
	// 从包头的 check digit 里拆出各把密钥的标识
	// ...
	err = sh.RetrieveSpecificSecret(secret)
	if err != nil {
		message.Error = err
		return
	}
	message.Secret = secret
	err = message.LoadBody(message.BodyBytes)
	return
}
```

包里附带一个基于内存的默认实现, 用两个 map 存 check digit 到密钥的映射, 加读写锁, 密钥更新走一个 channel 串行化. 够小规模场景直接用, 也是给使用方看的参考实现. 但它只是一个实现, 不是唯一的实现.

这个设计带来的好处比预想的多:

- **测试可以脱离密钥**. 注入一个固定密钥的 mock, 单元测试就不需要真的走密钥交换流程. 甚至可以注入一对 `MockEncryptBody` / `MockDecryptBody`, 让加密变成恒等函数, 联调时抓包能直接看到明文, 排查报文结构问题的效率完全不一样.
- **密钥的生命周期归业务**. 什么时候换密钥, 换失败了怎么退, 旧密钥留多久, 这些是业务策略, 写死在库里就没法改了.
- **库里不出现任何密钥**. 这个包从头到尾没有 "把密钥存哪" 这个概念, 自然也就不会有人往里面提交一个真的密钥. 这一点在合规审计的时候也少了很多麻烦.

反过来看, 接口开得有点多了. `RetrieveSecretWithCheckDigit` 和 `RetrieveSpecificSecret` 语义高度重合, 一个返回新对象一个原地填充, 是不同时期加的, 后来也没合并. 接口这东西, 一旦有人实现了就不好改了 —— 这大概是最直观的教训.

## SDK 化

上面这些东西 —— bitmap 怎么点位, 金额怎么压成 BCD, 商户名怎么转 Shift-JIS, 长度头算的是字节还是字符, 电文认证值用哪把密钥, 连接断了谁负责重连 —— 全都是**规格书的细节, 不是业务的细节**.

写这个 SDK 的时候我给自己定的目标是: 业务代码里不应该出现任何一个字节.

一笔授权交易, 在业务侧看起来应该长这样:

```go
req := client.ConstructMessage(
	models.MsgTypeCodeAuthorizationRequest,
	[]models.HeaderOption{
		models.WithSourceID(sourceID),
		models.WithDestinationID(destID),
	},
	[]models.BodyOption{
		models.WithF2(pan),
		models.WithF4(amount),
		models.WithFunctionCode(models.FunctionCodeAuthorization),
		models.WithF37(rrn),
		models.WithF43(merchantName), // 日文商户名, 直接传 UTF-8
	},
)
err = c.SendMessage(req)
```

到此为止. 至于这条报文的 bitmap 上点了哪几位, F4 被压成了 6 个字节, F43 被转成了 Shift-JIS, 包头上挂了一个用当前 KMAC 算出来的认证值, body 用当前 KC 加密过, 最后从连接池里挑了一条连接写出去 —— 业务不需要知道其中任何一件.

为了做到这一点, 包里的分层是这样的:

```text
      业务代码                        jcn/ligui (假中心 + 命令行)
          |                                    |
          +------------------+-----------------+
                             |
      ConstructMessage / SendMessage / ReceiveMessagesInChannel
                             |
+----------------------------------------------------------------+
| jcn/client          电文构造, 网络管理流程                     |
+----------------------------------------------------------------+
| jcn/center, pool    会话状态, 连接, 重连, 选路, 收发出入口     |
+----------------------------------------------------------------+
| jcn/models          这张网的报文定义, 密钥, 字段校验           |
+----------------------------------------------------------------+
| iso8583, cipher     通用封包解包, BCD / JIS8, 3DES             |
+----------------------------------------------------------------+
```

注意 `jcn/ligui` 的位置: 它在框外, 和业务代码平级. 它不是 SDK 的一层, 而是 SDK 的另一个使用者 —— 一个恰好扮演对端角色的应用. 依赖方向也是单向的: `ligui` 引用 `client` 和 `center`, 而 `client`, `center`, `models` 都不知道 `ligui` 的存在, 只有最上面的可执行程序才把它们拼在一起. 后面讲假中心的时候会回到这一点.

框里每一层只向上暴露上层真正需要的概念. 具体到几个决定:

**用 functional option 填域, 而不是暴露结构体.** body 上一共开了四十多个 `WithFxx`, 每个域一个. 这样业务代码是"我要填这几个域", 而不是"我要构造一个有这几个字段的结构体, 剩下的记得留 nil". 前者漏填一个域是编译能过但对端会拒的报文, 后者漏填一个域连自己都看不出来. 顺带, 这些 option 里可以做默认值和格式化, 比如交易时间直接 `models.TimeNumString(time.Now(), models.TimeFormatYYMMDDhhmmss)`, 业务不用记这个网络用的是 12 位还是 14 位时间.

**网络管理流程打包成一个调用.** 开局, 闭局, echo test, cutover, 密钥交换, 这些流程每个都是一来一回外加状态变更, 但业务侧只应该看到 `client.ConstructOpenRequest(...)` 和 `c.WaitForOnlineStatusOpened(30 * time.Second)`. 会话状态本身由库维护, 业务只关心"现在能不能发业务电文".

**日志要能直接看.** 加了一个反射实现的 `PrettyPrint`, 按域号顺序把非空的域打成 JSON, 二进制域转 hex, 原始字节不打:

```go
func (m *Message) JsonRawMessage() json.RawMessage {
	jsonRawMessage, _ := json.Marshal((*PrettyMessage)(m))
	return json.RawMessage(jsonRawMessage)
}
```

这件事看着不起眼, 但一个 SDK 好不好用, 很大程度上取决于出问题的时候能不能一眼看出是哪条报文的哪个域不对. 如果日志里只有一串十六进制, 那前面所有的抽象就没有意义了 —— 用的人还是得学会手动数字节.

反过来说, SDK 化最难的地方也在这: 你要在"藏起细节"和"出事的时候能查"之间找平衡. 藏得太浅, 业务代码里到处是规格书; 藏得太深, 出了问题只能来问你. 我的做法是**默认路径极简, 逃生舱口全开** —— `Message` 上原始的 `HeaderBytes` 和 `BodyBytes` 一直留着, 密钥, 状态, 连接池的统计也都能拿到, 只是正常写业务的时候用不上.

## 李逵与李鬼

包里有一个目录叫 `ligui`. 名字来自水浒里那个在树林里劫道, 自称"黑旋风李逵"的李鬼.

这就是开头说的那个挡板 —— 绕了一大圈, 最先要的东西反而是最后才做出来的.

理由倒是一直没变: 对端是别人的系统, 联调窗口按小时算, 要提前几周申请, 一次窗口跑不完就得等下一轮. 在这种节奏下, 唯一的办法是自己造一个假的中心, 平时先和它对接, 跑通了再去接真的.

`ligui` 就是这个假中心. 它做两件事:

**一是 stub, 扮演对端回应.** 收到什么请求就按规格书构造什么响应, 请求里该原样带回的域原样带回, 该由中心填的域 (承认号, 处理结果, 中心侧的交易通番) 由它来填:

```go
func constructAuthorizationResponse(req *models.Message, ...) (resp *models.Message) {
	resp = client.ConstructMessage(
		models.MsgTypeCodeAuthorizationResponse,
		[]models.HeaderOption{
			// 源和目的对调
			models.WithDestinationID(req.Header.SourceID.Value),
			models.WithSourceID(req.Header.DestID.Value),
		},
		[]models.BodyOption{
			models.WithF2(req.Body.F2.Value),
			models.WithF4(req.Body.F4.Value),
			models.WithF11(req.Body.F11.Value),
			// ...
			models.WithF38(""),        // to be filled by options
			models.WithActionCode(""), // to be filled by options
		},
	)
	resp.Update(headerOpts, bodyOpts)
	return
}
```

留空的那几个域, 由测试用例通过 option 覆盖. 于是同一个构造函数, 既能造一条"承认"的响应, 也能造一条"卡片无效"或者"限额超过"的响应. 异常路径的测试比正常路径重要得多, 因为正常路径联调时一定会过, 而异常路径可能上线半年才遇到第一次.

**二是 mock, 扮演对端的完整行为.** 这部分是一个状态机: 开局请求来了要回开局响应并且把在线状态推进到 opened, echo test 来了要回, cutover 来了要回, 密钥交换来了要用 KEK 解出新的工作密钥, 存进去, 再回一条响应. 业务电文来了就走 stub 那套构造响应. 遇到完全不认识的电文, 回一条通用错误通知, 并且把对方的包头原样贴在错误信息域里 —— 这是规格书要求的行为, 而这条路径如果没有 mock, 你几乎没有机会测到.

```go
default:
	logz.Debugf("unknown message received, message function: %v", message.MessageFunction())
	err = c.SendMessage(ConstructGeneralErrorMessageResponse(message, nil, []models.BodyOption{
		models.WithActionCode(models.ActionCodeErrorFormatInvalid),
		models.WithF72Byte(message.HeaderBytes),
	}), options...)
```

让双方跑同一个库, 还有一个更重要的理由: **假中心同时就是一个一致性检查器**. 我方发出去的报文, 假中心要能解开; 假中心发回来的报文, 我方要能解开. 任何一边对规格书的理解有偏差, 在自己和自己对接的时候就会暴露成一个解不开的报文, 而不是等到联调窗口里对方告诉你"你们的电文我们收不了".

在这个基础上又加了一层校验, 按电文种别和功能码, 声明这条报文**应该**出现哪些域:

```go
case models.MessageFunction(models.MessageTypeCodeNetworkControlResponse, models.FunctionCodeCutover):
	return CheckFiledByExpectedFileds(message, ExpectedFileds{[]int{11, 12, 24, 28, 39, ...}})
```

多一个域少一个域都算错. 加上每个域自己的规则校验 (卡号是 10 到 19 位数字, 处理码是 6 位且必须是规格书列出的组合之一), 等于把规格书里那张巨大的表格翻译成了可执行的断言. 这些断言在假中心里跑, 也在单元测试里跑.

### 在自己的笔记本上跑通全流程

李鬼是个库, 要用起来还得有个东西驱动它. 于是配了一个命令行, 一个最朴素的 REPL:

```go
func CommandLineRequest(c *center.Center, state *center.CommonState) (err error) {
	reader := bufio.NewReader(os.Stdin)
	for {
		fmt.Println("Ready for input......")
		input, err := reader.ReadString('\n')
		inputs := strings.Split(strings.TrimSuffix(input, "\n"), " ")
		err = DoRequest(c, state, inputs[0], inputs[1:]...)
		// ...
	}
}
```

命令基本覆盖了这张网上会发生的所有事:

```text
open                开局
echotest            心跳
key_exchange KC     换一把密钥 (KC / KMAC / KPE)
authori             一笔授权
authori_arqc        一笔带 IC 卡数据的授权
sales               一笔销售
revarsal_advice     一笔冲正
cutover <日期>       日切
audit               在线审查
stats               看看现在有几条连接
```

关键在于**两边跑的是同一个东西**. 最后落成两个 main, 一个 `center` 一个 `mockcenter`, 各自都只是几十行胶水: 读配置, 装好初始密钥, 起一个 `Center`, 然后一个把 stdin 接到 `ligui` 的 sender 上, 另一个用 `ligui` 的 receiver 自动应答. 地址填 `127.0.0.1`, 互相一连, 整条链路就在一台笔记本上跑起来了 —— 不需要专线, 不需要 VPN, 不需要对方的系统:

```text
open → key_exchange → authori → revarsal_advice → cutover → close
```

这一趟走下来, TCP 连接的建立与保活, 会话状态机的推进, bitmap 的组装与解析, BCD 和 Shift-JIS 的编解码, 3DES 加密, 电文认证值的计算与校验, 密钥按 check digit 的反查, 全都真实地执行了一遍. 它不是单元测试里那种"喂一段字节, 断言解出来是什么", 而是真的从一个进程的 socket 出去, 从另一个进程的 socket 进来 —— 一次完整的疏通测试, 只不过对面是自己.

它要解决的是下面几个问题.

**上手不需要任何前置条件.** 代码拉下来, 起两个进程就能跑起来. 不用申请专线, 不用等密钥下发, 不用协调联调窗口. 对一个外部依赖这么重的项目来说, "能在本机跑起来"这件事本身就把新人的上手成本砍掉了一大半 —— 否则一个刚接手的人在拿到网络权限之前, 连自己写的代码对不对都验证不了.

**报文的填充是自动的.** 一条授权报文二三十个域, 人不可能每次手填. 所以除了你真正关心的那一两个参数, 其余的域都用符合规格的随机数据填上 —— 检索参考号是合法字符集的 12 位, IC 卡数据是一段合法长度的字节:

```go
models.WithF37(jcn.MustGenerateCode("anp", 12)),
models.WithF55(jcn.MustGenerateByte("anp", 114)),
```

顺带, 这也一直在压边界. 每敲一次命令, 变长域的长度和内容都不一样, 连着敲几十次, 相当于一轮小小的随机测试 —— 前面那些长度头, 补位, 半角全角混排的坑, 有好几个就是这么撞出来的.

**有状态的流程终于可测了.** 密钥轮换这种事单元测试很难覆盖: 它跨越多条报文, 还要求双方的状态同步变化. 在命令行里就是一条 `key_exchange KC`, 敲完接着敲 `authori` —— 如果新密钥没生效, 对面立刻会因为认证值对不上把这笔拒掉, 一秒钟就知道结果.

**联调的时候人机配合很顺.** 对面在电话那头说"再发一笔带 IC 卡数据的", 敲一行就行, 不用改代码, 不用重新编译, 不用重新开局. 窗口时间按小时算的时候, 这个差别很大.

这个命令行本身的代码量很小, 大部分就是一个 switch 加一堆预设好的 option, 实现上没什么可说的. 但它是整个项目里被跑得最多的一段代码.

### 值不值

回头看, 这是整个项目里性价比最高的一部分. 报文结构的问题, 状态机的问题, 重连的问题, 九成都在和假中心对接的时候就暴露了. 真正的联调窗口于是只用来验证那些自己猜不到的东西 —— 比如对端的实现和规格书不一致的地方, 这种事总是有的, 而且往往是最后才发现.

写李鬼当然是有成本的, 粗看甚至像是把同一件事做了两遍. 但这类项目里对端不可控是常态: 你拿不到它, 约不到它, 也不能让它配合你重复一百次异常场景. 与其等着对端可用, 不如先把对端造出来.

## 后记

一个 SDK, 上千页日文 PDF. 目前全流程能跑通了, 不过后面付费联调和上线, 可能还要调整不少.

做对了的大概是三件事: 用反射加 struct tag 描述报文, 让 bitmap 不需要人来维护; 密钥用接口注入, 把存储和轮换的策略留给业务; 以及从第一天就写李鬼, 而不是等到有联调窗口了再说. 这三件事其实是同一件 —— 都是在把"某个人对规格书的理解"变成"代码里能跑, 能测, 能被别人改的东西". 这个包迟早不会由我来维护, 到那时候, 留下来的应该是断言和测试, 而不是口头传承.

而"业务代码里不应该出现任何一个字节"这条, 大概是我从这个项目里带走的最主要的东西 —— 一个 SDK 的价值不在于它实现了多少规格书, 而在于它替使用者挡掉了多少规格书.

做糙了的地方也不少. 连接池那部分改动最大, 根子上是状态的位置不一样: Redis 那边会话就绑在连接上, 连接本身是无状态, 可替换的, 池子里取哪一条都一样, 坏了丢掉重建就行; 而这边每条连接有自己的远端地址和身份, 会话又独立在连接之外, 池子能做的其实只剩下选路. 结构还是从 go-redis 借来的, 但里面的逻辑基本都换了一遍, 有些写法 (比如那个重试次数, 还有一处 `for` 循环里的提前 return) 现在看着就别扭. 密钥那几个接口开重复了, 也该趁早合并掉.

不过这大概就是这类项目的常态: 规格书是别人的, 时间是固定的, 能做的只有确保自己这部分完整可靠.
