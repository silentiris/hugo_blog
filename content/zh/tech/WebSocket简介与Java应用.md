+++
title = "WebSocket简介与Java应用"
date = "2023-06-18T14:32:49+08:00"
tags = []
slug = ""
+++

WebSocket简介与java实现

# http的不便

问题：我们经常会有前后端实时双向传递信息的需求，在需要实现实时信息传递的情境下，采用http应该怎么做呢？

方案：

1. 前端ajax轮询
2. http长轮询（http long poll）

这两种方法理论都可以实现，但却都不是那么方便。

首先来说说ajax轮询：

ajax轮询 的原理非常简单，让浏览器隔个几秒就发送一次请求，询问服务器是否有新信息。

接着看看http长轮询：

long poll 其实原理跟 ajax轮询 差不多，都是采用轮询的方式，不过采取的是阻塞模型（一直打电话，没收到就不挂电话），也就是说，客户端发起连接后，如果没消息，就一直不返回Response给客户端。直到有消息才返回，返回完之后，客户端再次建立连接，周而复始。

从上面两种方式能看出来，这两种方式都是不断地建立http连接并等待服务端处理，体现了http协议的一个特点**被动性** 。即服务端无法主动联系客户端，请求只能由服务端发起，后端如果有什么消息更新想告诉前端，是不方便的。

再来说说上面两种解决方案的问题所在：

ajax轮询 需要服务器有很快的处理速度和资源。

long poll 需要有很高的并发，也就是说同时接待客户的能力。

这就对服务器性能提出较高的要求，而且这两种实现方式都是非常消耗服务器资源的，例如对于ajax轮询来说，要不断地建立和断开http连接，很多时候还有查询数据库的需要，如果有https还需要校验证书，这都大大增加了服务器的性能压力。而且，http也是一种**无状态**的协议，无法记住之前的信息，这对于某些情境下也不是很方便，需要cookie或者session来处理。综上所述，HTTP 基于简单的**请求和响应模型**工作，这会产生很大的**延迟，**由于http在当前情景下的诸多不便，我们需要一些更加优雅的处理方式。

# Websocket

## 简介

WebSocket协议是基于TCP的一种新的网络协议。它实现了浏览器与服务器**全双工** （full-duplex）通信，即允许服务器主动发送信息给客户端。因此，在WebSocket中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输，客户端和服务器之间的数据交换变得更加简单。

WebSocket的握手连接基于http，但是在连接后就跟http没有关系，用封装好的基于tcp的方式来通信。

WebSocket协议不受同源策略影响。

这些特点让上述的问题有了一个比较好的解决方案。因为是全双工通信，服务端可以随时向客户端发信息，这使得后端如果有什么新的消息需要向前端汇报，前端可以第一时间接受，而且在后台占用的资源相较上边的两种方式而言低很多，处理速度快了很多。

![img](https://cdn.jsdelivr.net/gh/silentiris/pic_bed@latest/blog-images/1689659631020.PNG)

##   建立连接

- 第一步：客户端向服务端通过握手协议建立连接
- 第二步：服务端向客户端回应握手请求
- 第三步：服务端开始向客户端推送消息
- 第四步：客户端可以主动断开websocket连接

简单来说，ws首先会利用http的握手机制来进行连接。

### 客户端：申请协议升级

当客户端想要使用 WebSocket 协议与服务端进行通信时, 首先需要确定服务端是否支持 WebSocket 协议, 因此 WebSocket 协议的第一步是进行握手, WebSocket 握手采用 HTTP Upgrade 机制, 客户端可以发送如下所示的结构发起握手 (请注意 WebSocket 握手只允许使用 HTTP GET 方法)。

```HTTP
 GET / HTTP/1.1
 Host: localhost:8080
 Origin: http://127.0.0.1:3000
 Connection: Upgrade
 Upgrade: websocket、
 Sec-WebSocket-Version: 13
 Sec-WebSocket-Key: w4v7O6xFTi36lq3RNcgctw==
```

重点请求首部意义如下：

- `Connection: Upgrade`：表示要升级协议
- `Upgrade: websocket`：表示要升级到websocket协议。
- `Sec-WebSocket-Version: 13`：表示websocket的版本。如果服务端不支持该版本，需要返回一个`Sec-WebSocket-Version`header，里面包含服务端支持的版本号。
- `Sec-WebSocket-Key`：与后面服务端响应首部的`Sec-WebSocket-Accept`是配套的，提供基本的防护，比如恶意的连接，或者无意的连接。

> 注意，上面请求省略了部分非重点请求首部。由于是标准的HTTP请求，类似Host、Origin、Cookie等请求首部会照常发送。在握手阶段，可以通过相关请求首部进行 安全限制、权限校验等。

### **服务端：响应协议升级**

在 HTTP Header 中设置 Upgrade 字段, 其字段值为 websocket, 并在 Connection 字段指示 Upgrade, 服务端若支持 WebSocket 协议, 并同意握手, 可以返回如下所示的结构:

```HTTP
 HTTP/1.1 101 Switching Protocols
 Connection:Upgrade
 Upgrade: websocket
 Sec-WebSocket-Accept: Oy4NRAQ13jhfONC7bP8dTKb4PTU=
```

状态代码`101`表示协议切换。到此完成协议升级，后续的数据交互都按照新的协议来。

> 备注：每个header都以`\r\n`结尾，并且最后一行加上一个额外的空行`\r\n`。此外，服务端回应的HTTP状态码只能在握手阶段使用。过了握手阶段后，就只能采用特定的错误码。

#### **Sec-WebSocket-Accept的计算**

`Sec-WebSocket-Accept`根据客户端请求首部的`Sec-WebSocket-Key`计算出来。

计算公式为：

1. 将`Sec-WebSocket-Key`跟`258EAFA5-E914-47DA-95CA-C5AB0DC85B11`拼接。
2. 通过SHA1计算出摘要，并转成base64字符串。

伪代码如下：

```JavaScript
 >toBase64( sha1( Sec-WebSocket-Key + 258EAFA5-E914-47DA-95CA-C5AB0DC85B11 )  )
```

验证下前面的返回结果：

```JavaScript
 const crypto = require('crypto');
 const magic = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
 const secWebSocketKey = 'w4v7O6xFTi36lq3RNcgctw==';
 
 let secWebSocketAccept = crypto.createHash('sha1')
     .update(secWebSocketKey + magic)
     .digest('base64');
 
 console.log(secWebSocketAccept);
 // Oy4NRAQ13jhfONC7bP8dTKb4PTU=
```

## **状态码**

连接成功状态码

101：HTTP协议切换为WebSocket协议。

连接关闭状态码

1000：正常断开连接。

1001：服务器断开连接。

1002：websocket协议错误。

1003：客户端接受了不支持数据格式（只允许接受文本消息，不允许接受二进制数据，是客户端限制不接受二进制数据，而不是websocket协议不支持二进制数据）。

1006：异常关闭。

1007：客户端接受了无效数据格式（文本消息编码不是utf-8）。

1009：传输数据量过大。

1010：客户端终止连接。

1011：服务器终止连接。

1012：服务端正在重新启动。

1013：服务端临时终止。

1014：通过网关或代理请求服务器，服务器无法及时响应。

1015：TLS握手失败。

**连接关闭状态码是WebSocket对象的onclose属性返回的。**

## **心跳重连**

和真实的心跳一样，隔一段时间发一个小数据包，类似于ping pong！，用来判断连接是否还存在。

建议由客户端实现

# **WebSocket的Java实现**

## **依赖**

```XML
         <dependency>
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-websocket</artifactId>
         </dependency>
```

## **WebSocketConfig**

在`config`包新建`WebSocketConfig.java`，注入ServerEndpointExporter，这个bean会自动注册使用了@ServerEndpoint注解声明的Websocket endpoint。（要注意，如果使用独立的servlet容器，而不是直接使用springboot的内置容器，就不要注入ServerEndpointExporter，因为它将由容器自己提供和管理。）

```Java
 @Configuration
 public class WebSocketConfig {
     @Bean
     public ServerEndpointExporter serverEndpointExporter() {
         return new ServerEndpointExporter();
     }
 }
```

## WebsocketUtil

加入`@ServerEndpoint`和`@Component`注解。

`@ServerEndpoint` 注解是一个类层次的注解，它的功能主要是将目前的类定义成一个websocket服务器端, 注解的值将被用于监听用户连接的终端访问URL地址,客户端可以通过这个URL来连接到WebSocket服务器端。

加入`@Component`使其可以被spring容器扫描到。

```Java
package com.sipc.websocketdemo.websocketserver;

import jakarta.websocket.*;
import jakarta.websocket.server.PathParam;
import jakarta.websocket.server.ServerEndpoint;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@ServerEndpoint("/ws/{token}")
@Component
public class WebSocketUtil {
    private static int onlineCount = 0;//在线人数
    private static ConcurrentHashMap<String, WebSocketUtil> webSocketMap = new ConcurrentHashMap<>();//在线用户集合
    private Session session;//与某个客户端的连接会话
    private String currentUser;

    /**
     * 获取当前所有在线用户名
     */
    public static void allCurrentOnline() {
        for (Map.Entry<String, WebSocketUtil> item : webSocketMap.entrySet()) {
            System.out.println(item.getKey());
        }
    }

    /**
     * 发送给指定用户消息
     */
    public static void sendMessageTo(String message, String token) {
        WebSocketUtil item = webSocketMap.get(token);
        System.out.println("to"+token+":" + message);
        try {
            item.session.getBasicRemote().sendText(message);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 群发自定义消息
     */
    public static void sendInfo(String message) {
        System.out.println(message);
        for (Map.Entry<String, WebSocketUtil> item : webSocketMap.entrySet()) {
            item.getValue().sendMessage(message);
        }
    }

    public static synchronized int getOnlineCount() {
        return onlineCount;
    }

    public static synchronized void addOnlineCount() {
        WebSocketUtil.onlineCount++;
    }

    public static synchronized void subOnlineCount() {
        WebSocketUtil.onlineCount--;
    }

    @OnOpen         //有新连接时触发
    public void onOpen(@PathParam("token") String token, Session session) {
        this.currentUser = token;
        this.session = session;
        webSocketMap.put(token, this);
        addOnlineCount();
        System.out.println("有新连接" + currentUser + "加入！当前在线人数为" + getOnlineCount());
    }

    @OnClose        //有连接关闭时触发 
    public void onClose() {
        String closeUser = this.currentUser;
        webSocketMap.remove(this.currentUser);
        subOnlineCount();
        System.out.println(closeUser + "连接关闭！当前在线人数为" + getOnlineCount());
    }

    @OnMessage        //有连接传来的新消息时触发
    public void onMessage(String message, Session session) {
        System.out.println("来自客户端"+currentUser+"的消息：" + message);
        for (Map.Entry<String, WebSocketUtil> item : webSocketMap.entrySet()) {
            if (item.getValue() == this) {
                sendMessageTo("cnm", this.currentUser);
                continue;
            }
            item.getValue().sendMessage(message);
        }
    }

    @OnError        
    public void onError(Session session, Throwable throwable) {
        System.out.println("发生错误！");
        throwable.printStackTrace();
    }
    /**
     * 给自己发送自己刚发的消息
     */
    public void sendMessage(String message) {
        try {
            this.session.getBasicRemote().sendText(message);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

}
```

参考:

https://www.cnblogs.com/chyingp/p/websocket-deep-in.html

https://zhuanlan.zhihu.com/p/145628937

https://sunyunqiang.com/blog/websocket_protocol_rfc6455/