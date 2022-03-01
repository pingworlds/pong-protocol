# pong-protocol

pong protocol 是一个支持在单一网络连接上多路并发的代理协议，分为local 和remote 端两部分运行。


## 背景

当前流行的代理协议基本上都源出socks5，基本特征是一个网络连接承载一个代理请求，
这种模式存在着无法克服的“连接阻塞”缺陷，具体表现为：
 
  - 浪费连接数，浏览一个新闻/图片网站，可能需要一次性消耗50条以上的网络连接
  - 浪费带宽，连接数限制严重制约了本地和代理服务端的网络带宽利用空间
  - 卡顿，前置web server会限制每个客户端最大连接数，连接数配额耗尽后，客户端会卡顿至无法服务，
    必须等待连接超时，或者强制重启客户端来释放连接
  - 延迟，每一个代理请求都需要时间去打开网络连接
  - 空耗，被打开的网络连接99.9%的时间里处于空闲状态


pong协议通过多路复用的方式减少连接数来规避“连接阻塞”现象，当然这很可能会引发其它未知问题。 


## pong 简介

pong协议采用http2模式，可以在一条网络连接上承载数十数百条并发代理请求，它是socks5和http2的结合体，特征：

  - 高并发，通常情况下，1条网络连接支持所有代理请求
  - 0 open time，收到代理请求后利用已经建立的网络连接通道立即转发出去，不需要open time
  - 高吞吐量，可以将代理服务的流量转发马力最大化和支持更多的客户端
  - 传输协议无关，支持主流互联网标准协议
  - 明文，数据加密依赖于标准的传输协议，减少协议实现的复杂性
  - 开放，任何人都可以自由使用
  - 扩展性强，容易扩展私有特征，比如强化用户验证


 
## stream

和http2一样，pong使用stream来表示local和remote之间的数据交互序列

stream 是双向的，local与remote在同一个流上互发数据。

stream 是并发的，无需等待一个流结束才开始下一个流，并发数量没有限制，由具体实现决定。

stream 由若干个 frame 数据包组成。

###  frame

frame 由必须的header和可选的paylaod组成， 结构如下：

    +---------+-------------+------------+-------------++
    |   len   |  stream id  | frame type |   payload    |
    | 2 byte  |    4  byte  |   1 byte   |  0...n byte  |           
    +---------+-------------+------------+--------------+

payload 是长度为[0,65500]的二进制数据，可能是data frame的转发数据，也可以是command frame的附加信息

frame是有序的，但它本身不包含序号，顺序由底层传输协议自动保证。


#### frame header

header 固定7个字节，len(2字节) + stream id(4字节) + frame type (1 字节)


- len  16位整数，2个字节，大端序，表示整个frame数据包总长度，包括长度字段本身
- stream id  32位整数，4个字节，大端序，标识流.
  
  local端发起的 stream 的 id 为奇数.

  remote端发起的 stream 的 id 位偶数，虽然可能永远不会发起。id从1开始是个好习惯。
- frame type   1 个字节，区分不同类型的数据包


当前定义的frame type :
 
    0x00   data        //data 
    0x01   connnect    //socks5 connect 
    0x02   bind        //socks5 bind 
    0x03   assocaiate  //socks5 udp assocaiate 
    0x05   relay       //udp relay 
    0x06   set         //settings  
    0x07   finish      //close send channel,half close
    0x08   close       //close stream
    0x09   rst         //break stream


0x00 data之外的frame type 统称为command frame


#### 0x00 data  

payload 负载是需要转发的实际有效数据


#### 0x01 connect  

0x01 表示 socks5 connect 请求，payload是一个标准的socks5地址

    +----+-----+-------+------+----+ 
    | ATYP | DST.ADDR  |  DST.PORT |
    +----+-----+-------+------+----+ 
    |  1   | Variable  |    2      |
    +----+-----+-------+------+----+ 


#### 0x02 bind  （未定义）

socks5 bind  应遵循socks5标准


#### 0x03 udp assocaiate  （未定义）

socks5 udp assocaiate   应遵循socks5标准

#### 0x05 udp relay   

udp relay 透明转发udp 数据，payload为一个socks5 地址，等价于一个connect, 只是标识为udp以通知remote按udp方式处理.

与udp assocaiate的区别是，它不会建立udp通道，也不关心remote怎么处理，只是将udp请求和数据全权委托给remote


#### 0x06 set   

保留


#### 0x07 finish   

通知对方peer 数据发送完毕，关闭发送通道


#### 0x08 close   

通知对方peer 流处理完毕，关闭流

 
#### 0x09 rst   

通知对方peer 流异常终止， 忽略后续数据包，立即关闭它。

rst 可以不包含错误码,没有payload，也可以在payload中附加错误码和错误描述，第一个字节为错误码，后续字节是描述。

错误码可以帮助对方peer分析错误，沿用并扩展了socks5定义的错误，已定义的错误码列表：

    //socks5 error code
    ERR                 byte = 1     "proxy server error",
    ERR_RULE            byte = 2     "proxy server rule refuse",
    ERR_NET             byte = 3     "network unreachable"
    ERR_DST             byte = 4     "host unreachable",
    ERR_DST_REFUSE      byte = 5     "refused byte dst",
    ERR_TTL_EXPIRED     byte = 6     "TTL expired",
    ERR_COMMAND_UNKNOWN byte = 7     "command not supported",
    ERR_ADDR_TYPE       byte = 8     "address type not supported",

    //pong 扩展
    ERR_REPEAT_ID       byte = 9     "repeat stream id",
    ERR_DIAL_TIMEOUT    byte = 10    "dial dst i/o timeout",
    ERR_DST_RESET       byte = 11    "reset by peer",
    ERR_CONN_TIMEOUT    byte = 12    "forcibly closed by the remote host",
 


#### 扩展

两种方式，
- 扩展frame type，
- 自定义finish, rst ,close 的payload数据结构


## 交互

pong 网络连接的交互过程如下图

数字表示stream id

    local                                remote 
    |     --- connection header   --->    | 
    |     ---   method connect 1  --->    | 创建 stream 1
    |     ---   method connect 3  --->    | 创建 stream 3
    |     ---   method connect 5  --->    | 创建 stream 5
    |     ---     data  3         --->    | 发送 stream 3 data
    |     ---     data  1         --->    | 发送 stream 1 data
    |     ---     data  5         --->    | 发送 stream 5 data
    |     ---     data  1         --->    | 发送 stream 1 data
    |     ---     finish  1       --->    | 关闭 stream 1 发送通道
    |     ---     finish  3       --->    | 关闭 stream 3 发送通道
    |     <---    data  5         ----    | 响应 stream 5 data 
    |     <---    data  3         ----    | 响应 stream 3 data
    |     ---     rst  3          --->    | 取消 stream 3 
    |     <---    finish 5        ----    | 关闭 stream 5 响应通道
    |     <---    ...             ----    | ....


### 建立连接通道


local 打开remote连接后，首先发送一个用于确定身份的连接头部;

remote 收到头部后验证合法性，合法则维持连接，等待后续数据包，不合法则直接关闭连接。

连接头部17个字节，1字节的版本号，默认为0，16个字节合法的uuid形式的client id， 

    --------------------------+ 
    |   ver    | client id    |
    +----+-----+-------+------+ 
    |  1 byte  |   16 byte    |
    --------------------------+ 

###  创建流

发送 connect,bind,associate,relay之一来创建stream

- local收到本地软件src(browser,vpn...)的代理请求后，将请求dst地址封装成connect frame发送给remote，
   
  无需等待remote响应，remote可能根本不会响应，继续发送后续data frame；
- remote 收到 connect frame后，读取payload中的socks5地址，打开目标服务dst的网络连接，建立转发通道，
  如果打开失败，向local发送一个rst frame 终止流.
   
  如果成功，顺次取出local发送来的后续data frame中 payload 发送给dst，
  并将dst的响应数据封装成data frame 回送给local端, 


如此这般，src <--> local <---> remote <---> dst 之间的网络通道建立。


##  传输协议

pong 不依赖于具体传输协议，支持tcp,tls,http,https,http2,h2c,h3,ws,wss,grpc，what ever...

pong 本身是明文，不支持加密，加密不应该是代理协议的功能，代理协议在加密上下苦功是没有意义的重复劳动，
数据加密应该依赖底层传输协议，比如 tls tcp,https,http2,http3,wss

 


 ## pong 实现


   - pong-go  
     golang 实现，<https://github.com/pingworlds/pong>


 ## pong 客户端

   - ping  
    支持Android，<https://github.com/pingworlds/ping>

 
 


