# pong-protocol

- [中文](README.md)
- [English](pong_en.md)


The pong protocol is a proxy protocol that supports multiple concurrent connections on a single network connection, running in two parts: the local and remote side.


## Background

The current popular proxy protocols basically originate from socks5, with the basic feature that one network connection carries one proxy request.
This model has the insurmountable defect of "connection blocking", which is manifested as follows.
 
  - Wasted connections, browsing a news/photo site may require more than 50 network connections at once
  - Wasted bandwidth, the connection limit severely restricts the network bandwidth utilization of the local and proxy servers
  - Stalling, the front web server will limit the maximum number of connections per client, after the connection quota is exhausted, the client will stall until it is unable to serve.
    You must wait for the connection to time out or force a restart of the client to release the connection.
  - Delay, each proxy request needs time to open the network connection
  - Idle, open network connections are idle 99.9% of the time


The pong protocol circumvents the "connection blocking" phenomenon by reducing the number of connections through multiplexing, but this may cause other unknown problems. 


## Introduction to pong

The pong protocol uses the http2 model, which can host tens of hundreds of concurrent proxy requests on a single network connection, and is a combination of socks5 and http2, featuring

  - High concurrency, usually, 1 network connection supports all proxy requests
  - 0 open time, after receiving a proxy request, it is immediately forwarded out using the established network connection channel, no open time is needed
  - High throughput, can maximize the traffic forwarding horsepower of the proxy service and support more clients
  - Transport protocol independent, support mainstream Internet standard protocols
  - Explicit, data encryption relies on standard transmission protocols, reducing the complexity of protocol implementation
  - Open, free for anyone to use
  - Extensible, easy to extend private features, such as enhanced user authentication

## stream

As with http2, pong uses stream to represent the sequence of data interaction between local and remote

stream is bidirectional, local and remote send data to each other on the same stream.

stream is concurrent, no need to wait for one stream to finish before starting the next stream, there is no limit to the number of concurrent streams, it is implementation specific.

A stream consists of a number of frame packets.

### frame

A frame consists of a mandatory header and an optional paylaod, with the following structure.

    +---------+-------------+------------+-------------++
    | len | stream id | frame type | payload |
    | 2 byte | 4 byte | 1 byte | 0.... .n byte |           
    + ---------+-------------+------------+--------------+

payload is a binary data of length [0,65500], which may be the forwarding data of a data frame, or it may be additional information of a command frame

The frame is ordered, but it does not contain a sequential number itself; the order is automatically guaranteed by the underlying transport protocol.


#### frame header

header fixed 7 bytes, len (2 bytes) + stream id (4 bytes) + frame type (1 byte)


- len 16-bit integer, 2 bytes, large end order, indicating the total length of the entire frame packet, including the length field itself
- stream id 32-bit integer, 4 bytes, big-endian, identifies the stream.
  
  The id of the stream initiated on the local side is odd, and it is a good habit to start with 1.

  The remote side initiates a stream with an even id, although it may never be initiated.
- frame type 1 byte to distinguish between different types of packets


The currently defined frame type :

 
    0x00   data        //data 
    0x01   connnect    //socks5 connect 
    0x02   bind        //socks5 bind 
    0x03   associate  //socks5 udp associate 
    0x05   relay       //udp relay 
    0x06   set         //settings  
    0x07   finish      //close send channel,half close
    0x08   close       //close stream
    0x09   rst         //break stream

0x00 frame types other than data are collectively referred to as command frames


#### 0x00 data  

payload is the actual valid data that needs to be forwarded


#### 0x01 connect  

0x01 means socks5 connect request, payload is a standard socks5 address

    +----+-----+-------+------+----+ 
    | ATYP | DST.ADDR | DST.PORT |
    +----+-----+-------+------+----+ 
    | 1 | Variable | 2 |
    +----+-----+-------+------+----+ 


#### 0x02 bind (undefined)

socks5 bind should follow the socks5 standard


#### 0x03 udp associate (undefined)

socks5 udp associate should follow the socks5 standard

#### 0x05 udp relay   

udp relay transparently forwards udp data, payload is a socks5 address, equivalent to connect, except that it is identified as udp to inform the remote that it is being handled as udp.

The difference with udp associate is that it doesn't create a udp channel and doesn't care what remote does with it, it just delegates the udp request and data to remote


#### 0x06 set   

set


#### 0x07 finish   

Notify the other peer that the data has been sent and close the sending channel


#### 0x08 close   

Notify the other peer that the stream is finished, close the stream

 
#### 0x09 rst   

Notify the other peer that the stream is abnormally terminated, ignore subsequent packets, and close it immediately.

The rst can contain no error code, no payload, or the error code and error description can be appended to the payload, the first byte is the error code and the subsequent bytes are the description.

The error code can help the other peer to analyze the error, following and extending the error defined by socks5, the list of defined error codes.


    //socks5 error code
    ERR byte = 1 "proxy server error",
    ERR_RULE byte = 2 "proxy server rule refuse",
    ERR_NET byte = 3 "network unreachable", ERR_RULE byte = 2 "proxy server rule refuse", ERR_NET byte = 3 "network unreachable"
    ERR_DST byte = 4 "host unreachable",
    ERR_DST_REFUSE byte = 5 "refused byte dst",
    ERR_TTL_EXPIRED byte = 6 "TTL expired",
    ERR_COMMAND_UNKNOWN byte = 7 "command not supported",
    ERR_ADDR_TYPE byte = 8 "address type not supported",

    //pong extension
    ERR_REPEAT_ID byte = 9 "repeat stream id",
    ERR_DIAL_TIMEOUT byte = 10 "dial dst i/o timeout",
    ERR_DST_RESET byte = 11 "reset by peer",
    ERR_CONN_TIMEOUT byte = 12 "forcibly closed by the remote host",
 


#### extension

Both ways.
- Extend frame type.
- Customize the payload data structure for finish, rst , close


## Interaction

The interaction process of pong network connection is as follows

The number indicates the stream id

    local remote 
    | --- connection header ---> | 
    |-- method connect 1 ---> | create stream 1
    | --- method connect 3 ---> | create stream 3
    | --- method connect 5 ---> | create stream 5
    | --- data 3 ---> | send stream 3 data
    | --- data 1 ---> | send stream 1 data
    | --- data 5 ---> | send stream 5 data
    | --- data 1 ---> | send stream 1 data
    | --- finish 1 ---> | close stream 1 send channel
    | --- finish 3 ---> | close stream 3 send channel
    | <-- data 5 ---- | respond to stream 5 data 
    | <-- data 3 ---- | response stream 3 data
    | --- rst 3 ---> | cancel stream 3 
    | <-- finish 5 ---- | close stream 5 response channel
    | <-- ...             ---- | ....


### Establishing a connection channel


local opens a remote connection and first sends a connection header to determine the identity of the connection;

remote receives the header and verifies the legitimacy, if it is legal, the connection is maintained and waits for subsequent packets, if not, the connection is closed.

Connection header is 17 bytes, 1 byte version number, default is 0, 16 bytes client id in the form of legal uuid. 

    --------------------------+ 
    | ver | client id |
    + ----+-----+-------+------+ 
    | 1 byte | 16 byte |
    --------------------------+ 

### Create stream

Send one of connect,bind,associate,relay to create stream

- local receives a proxy request from the local software src(browser,vpn...) After receiving the proxy request from the local software src(browser,vpn...), it wraps the request dst address into a connect frame and sends it to the remote.
   
  It does not have to wait for a response from remote, which may not respond at all, and continues to send subsequent data frames.
- After receiving the connect frame, the remote reads the socks5 address in the payload, opens the network connection of the target service dst, and establishes a forwarding channel.
  If it fails to open, it sends an rst frame to the local to terminate the flow.
   
  If it succeeds, take out the payload in the subsequent data frame sent by local and send it to dst.
  and wraps the response data from dst into a data frame and sends it back to local, 


In this way, the network channel between src <--> local <--> remote <--> dst is established.


## Transport protocols

pong does not depend on a specific transport protocol, it supports tcp,tls,http,https,http2,h2c,h3,ws,wss,grpc,what ever...

pong itself is plaintext, does not support encryption, encryption should not be the function of the proxy protocol, the proxy protocol in the encryption of hard work is meaningless duplication of effort.
Data encryption should rely on the underlying transport protocols, such as tls tcp,https,http2,http3,wss

 

 ## pong implementation


   - pong-go  
     golang implementation, <https://github.com/pingworlds/pong>


 ## pong client

   - ping  
    Android support, <https://github.com/pingworlds/ping>

 
 
