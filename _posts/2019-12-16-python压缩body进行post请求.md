# python压缩body进行post请求

标签（空格分隔）： python
---
最近工作中需要对access日志进行上报，由于日志量较大，所以上报时进行了压缩处理，将日志信息放在请求body中，再进行zlib压缩处理，可以较好提升性能

## 使用requests实现长连接
因为要对日志进行逐条上报，单个body请求大小有限，所以要采用长连接。这里使用了requests库，在该库中，HTTP的长连接是通过Session会话实现的。官方对Session库的说明如下：

>会话对象让你能够跨请求保持某些参数。它也会在同一个 Session 实例发出的所有请求之间保持 cookie， 期间使用 urllib3 的 connection pooling 功能。所以如果你向同一主机发送多个请求，底层的 TCP 连接将会被重用，从而带来显著的性能提升。
HTTP persistent connection, also called HTTP keep-alive, or HTTP connection reuse, is the idea of using a single TCP connection to send and receive multiple HTTP requests/responses, as opposed to opening a new connection for every single request/response pair. The newer HTTP/2 protocol uses the same idea and takes it further to allow multiple concurrent requests/responses to be multiplexed over a single connection.

实现代码：
```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

import requests
import time

url="http://172.20.29.116:5140"

payload=[{'mesg': 'one'},{'mesg':'two'}]

s = requests.Session()
r = s.post(url, data=payload)
print r.headers
print r.status_code
print r.content

while True:
    time.sleep(10)
```

使用Session发送HTTP请求后，连接依然是ESTABLISHED。
如果不适用Session，发送请求后，连接马上断开，下次请求又要重新建立连接，这样带来更高的耗时，浪费CPU资源。

## 压缩HTTP请求正文
### 使用gzip库
使用gzip库需要使用StringIO模块的StiongIO类，创建StringIO对象当做一个文件，传给gzip的GzipFile 对象，GzipFile会将字符串压缩后写入StringIO对象。使用 getvalue方法从StringIO中获取压缩后的字符串
```
s = StringIO.StringIO()
g = gzip.GzipFile(fileobj=s, mode='w')
g.write(post_data)
g.close()
gzipped_data = s.getvalue()
```

### 使用zlib库
使用zlib库非常简单，直接调用compress函数进行压缩，调用decompress进行解压。参数string指定了要压缩的数据流，参数level指定了压缩的级别，它的取值范围是1到9。压缩速度与压缩率成反比，1表示压缩速度最快，而压缩率最低，而9则表示压缩速度最慢但压缩率最高:
```
import zlib
message = 'abcd1234'
compressed = zlib.compress(message)
decompressed = zlib.decompress(compressed)

print 'original:', repr(message)
print 'compressed:', repr(compressed)
print 'decompressed:', repr(decompressed)
```
结果
```
original: 'abcd1234'
compressed: 'x\x9cKLJN1426\x01\x00\x0b\xf8\x02U'
decompressed: 'abcd1234'
```


