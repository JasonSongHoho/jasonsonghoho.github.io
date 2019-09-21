---
title: WebSocket 快速上手
date: 2019-09-21 18:58:29
tags:
- WebSocket
---

## 简介
本文介绍如何基于 SpringBoot 快速搭建 WebSocket 服务端和客户端。

### WebSocket 使用场景
与 Http 协议相比，WebSocket 有两大优势：
1. 支持服务端主动向客户端推送消息，不需要客户端进行轮询服务端。
2. 节省网络带宽。维持一个长连接，客户端和服务端通信不需要频繁的建立连接。且互相沟通的Header非常小。

## Quick Start
开始搞起
### maven 依赖

```
<!--服务端依赖-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
    <version>2.0.6.RELEASE</version> <!--版本自选-->
</dependency>

<!--客户端依赖-->
<dependency>
    <groupId>org.java-websocket</groupId>
    <artifactId>Java-WebSocket</artifactId>
    <version>1.4.0</version> <!--版本自选-->
</dependency>
```

### 服务端代码
新建一个 SpringBoot 应用。添加 MyWebSocket 类。

```
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.websocket.*;
import javax.websocket.server.PathParam;
import javax.websocket.server.ServerEndpoint;
import java.io.IOException;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
@Component
@ServerEndpoint("/myWebSocket/{id}")
public class MyWebSocket {

    public static String heartBeat = "HeartBeat";
    public static ConcurrentHashMap<Long, MyWebSocket> webSocketMap = new ConcurrentHashMap<>();

    /**
     * 与某个客户端的连接会话，需要通过它来与客户端进行数据收发
     */
    private Session session;
    private Long id;

    public static int getOnlineCount() {
        return webSocketMap.size();
    }


    @OnOpen
    public void onOpen(Session session, @PathParam("id") Long id) {
        log.info("Open a webSocket. id={}", id);
        this.session = session;
        this.id = id;
        webSocketMap.put(id, this);
    }

    @OnClose
    public void onClose() {
        webSocketMap.remove(id);
        log.info("Close a webSocket. id:{}", id);
    }

    @OnMessage
    public void onMessage(String message, Session session) {
        if (heartBeat.equals(message)) {
            try {
                session.getBasicRemote().sendText(message);
                log.debug("Receive a heartBeat message from client: {}. id:{}", message, id);
                return;
            } catch (IOException e) {
                log.error("回复心跳信息失败！", e);
            }
        }
        log.info("Receive a message from client: {}. id:{}", message, id);
    }

    @OnError
    public void onError(Session session, Throwable error) {
        log.error("Error while webSocket. ", error);
    }


    public Session getSession() {
        return session;
    }

    public Long getId() {
        return id;
    }

}
```

### 客户端代码

```
import lombok.extern.slf4j.Slf4j;
import org.java_websocket.client.WebSocketClient;
import org.java_websocket.drafts.Draft;
import org.java_websocket.drafts.Draft_6455;
import org.java_websocket.handshake.ServerHandshake;

import java.net.URI;
import java.net.URISyntaxException;
import java.util.concurrent.TimeUnit;

@Slf4j
public class TestWebSocketClient extends WebSocketClient {
    public static String heartBeat = "HeartBeat";


    public TestWebSocketClient(URI serverUri, Draft protocolDraft) {
        super(serverUri, protocolDraft);
    }

    public static void main(String[] args) throws URISyntaxException, InterruptedException {
        //端口号与 SpringBoot 的 servlet 容器端口一致
        String serverUrl = "ws://localhost:9292/myWebSocket/2000000000462815";
        URI recognizeUri = new URI(serverUrl);
        TestWebSocketClient client = new TestWebSocketClient(recognizeUri, new Draft_6455());
        client.connectBlocking(5, TimeUnit.SECONDS);
        client.send("This is a message from client. ");
        while (true) {
            client.send(heartBeat);
            Thread.sleep(2000);
        }
    }

    @Override
    public void onOpen(ServerHandshake serverHandshake) {
        log.info("Open a WebSocket connection on client. ");
    }

    @Override
    public void onClose(int code, String reason, boolean remote) {
        log.info("Close a WebSocket connection on client.  {} {} {}", code, reason, remote);
    }

    @Override
    public void onMessage(String message) {
        log.info("WebSocketClient receives a message: " + message);
    }

    @Override
    public void onError(Exception exception) {
        log.error("WebSocketClient exception. ", exception);
    }

}
```




### WebSocketConfig
用 SpringBoot 运行应用时，需要再添加一个配置文件，将 ServerEndpointExporter 注入 Bean 容器。
```
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.socket.server.standard.ServerEndpointExporter;

/**
 * 用spring boot运行应用时，打开 @Configuration 注释；使用 war 部署在tomcat时，关闭注释
 *
 */

@Configuration
@Slf4j
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```


### 结语
搞定，好快～



