---
title: spring-websocket实现聊天室功能
date: '2022/9/8 22:48'
swiper: true
swiperImg: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-57o7o1.jpg'
cover: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-57o7o1.jpg'
categories: WebSocket
tags:
  - spring
abbrlink: 9748f1b4
---

# spring-websocket实现聊天室功能

最近看到有些人的博客中有聊天室的功能所以我也在我博客中写了一个，不过他们用的是java原生的，这里我使用了spring封装的spring-websocket

## Spring-WebSocket配置

我们第一步要先配置一下websocket 的基本信息

```java
/**
 * @Author: ZVerify
 * @Description: TODO WebSocket相关配置
 * @DateTime: 2022/9/6 14:21
 **/
@Configuration
@EnableWebSocket
public class ZVerifyWebSocketConfig implements WebSocketConfigurer {

    // 注册 WebSocket 处理器
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry webSocketHandlerRegistry) {
        webSocketHandlerRegistry
                // WebSocket 连接处理器
                .addHandler(new ZVerifyWebSocketHandler(), "/ws-connect")
                // WebSocket 拦截器
                .addInterceptors(new ZVerifyWebSocketInterceptor())
                // 允许跨域
                .setAllowedOrigins("*");
    }

}
```

>其中连接处理器和拦截器是我们自己定义的

`"/ws-connect"`就是我们的路径

因为想要建立连接首先要通过我们的拦截器所以按照逻辑来写拦截器

## 前置拦截器

这个前置拦截器一般我们会做安全的校验和一系列处理，这里我就简单了写了一下，这里要做安全校验是因为我们定义的websocket并没有托管给我所使用的安全框架去验证用户，所以在这里要简单校验一下，

前置处理器的创建要去实现HandshakeInterceptor接口然后重写beforeHandshake，afterHandshake，两个方法，beforeHandshake是用做握手前置校验的，afterHandshake是做握手后置校验的

```java
/**
 * @Author: ZVerify
 * @Description: TODO WebSocket 前置拦截器
 * @DateTime: 2022/9/6 14:19
 **/
@Configuration
public class ZVerifyWebSocketInterceptor implements HandshakeInterceptor {
    // 握手之前触发 (return true 才会握手成功 )
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler handler,
                                   Map<String, Object> attr) {

        System.out.println("---- 握手之前触发 " + StpUtil.getTokenValue());

        // 未登录情况下拒绝握手
        if(!StpUtil.isLogin()) {
            System.out.println("---- 未授权客户端，连接失败");
            return false;
        }

        // 标记 userId，握手成功
        attr.put("userId", StpUtil.getLoginIdAsLong());
        
        return true;
    }

    // 握手之后触发
    @Override
    public void afterHandshake(ServerHttpRequest request, ServerHttpResponse response, WebSocketHandler wsHandler,
                               Exception exception) {
        System.out.println("---- 握手之后触发 ");
    }

}
```

## 连接处理器

这里是我们的主要处理器，基本上所有重要业务都在这里

首先创建一个自己的ZVerifyWebSocketHandler然后再去继承TextWebSocketHandler我们可以定制的去实现里边的方法，这里我就按照我自己的博客需求进行重写了，如果需要可以自行扩展。

![image-20220908205224501](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220908205224501.png)

### 重要属性

![image-20220908205537147](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220908205537147.png)

这个是用来存放我们当前在线的人的信息的，用于广播和人数统计还有私信

### 进入聊天成功的逻辑

首先重写afterConnectionEstablished()方法这个方法是在连接开启的时候触发的，也就是我握手成功之后，因为是聊天室所以功能防QQ做了，在登录之后会看到当前博客群聊中的在线人数，然后加载聊天记录。这一些简单的过程

![image-20220908211059504](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220908211059504.png)

0. 首先要从session中取到当前连接的用户id，这里我要解释一下这个userId是从哪来的，是在我的握手之前触发的那个beforeHandshake()中写的项目中用的安全框架为Sa-Token，不熟悉的请自行查阅，拿到用户id之后将当前用户的webSocketSession存放到map中

1. 更新当前的在线人数，这个处理是比较简单的![image-20220908212420367](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220908212420367.png)

   就是获取一下map的大小就是当前在线人数，然后发送广播消息，这里说一下广播消息其实很简单就是将map中的webSocketSession都取出来然后挨个发送消息注意这里要加一个锁因为不加锁的话可能会导致消息前后异常![](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220908213052294.png)

2. 加载历史记录也很平常就是将我们聊天记录存到数据库中，然后将其xxx小时的消息加载出来，然后想当前登录用户发送这里我使用的是历史12小时![image-20220908213511483](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220908213511483.png)

### 收到消息之后处理逻辑

处理收到消息逻辑是handleTextMessage()方法里边有两个参数一个是发送消息的session，一个是包装的消息对象TextMessage，首先先带大家看一下TextMessage是个什么东西，我们在通过webSocketSession发送消息的时候可以发送多种对象![image-20220908214150426](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220908214150426.png)

这里我使用了TextMessage，所以就讲一下这里我们在创建TextMessage对象的时候传入参数通过源码可以知道我可以传入一个可读的char值序列然后会将其转换成字符串调用抽象类的构造方法![image-20220908214411389](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220908214411389.png)

第二个参数的意义是这是否是作为一系列部分消息发送的消息的最后一部分。到这里可以知道我们发送的消息就是抽象类AbstractWebSocketMessage中的payload属性，所以在这里我买可以通过这个入参拿到数据，然后根据其数据的第一个参数，也就是当前的类型去进行对应的逻辑处理，这里就没什么难点了

### 连接关闭

![image-20220908214928823](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20220908214928823.png)

连接关闭的时候讲当前的用户session从map中remove掉就好如需扩展请自己进行逻辑的修改

### 源码

```java
package com.zang.blogz.handler;

import cn.hutool.core.date.DateUtil;
import cn.hutool.json.JSONUtil;
import com.alibaba.fastjson.JSON;
import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.zang.blogz.dto.ChatRecordDTO;
import com.zang.blogz.dto.RecallMessageDTO;
import com.zang.blogz.dto.WebsocketMessageDTO;
import com.zang.blogz.enmus.ChatTypeEnum;
import com.zang.blogz.enmus.FilePathEnum;
import com.zang.blogz.entity.ChatRecord;
import com.zang.blogz.entity.UserInfo;
import com.zang.blogz.model.input.ro.VoiceRO;
import com.zang.blogz.service.ChatRecordService;
import com.zang.blogz.service.UserInfoService;
import com.zang.blogz.steam.optional.Opp;
import com.zang.blogz.strategy.context.UploadStrategyContext;
import com.zang.blogz.utils.BeanCopyUtils;
import com.zang.blogz.utils.IpUtil;
import lombok.Data;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.socket.CloseStatus;
import org.springframework.web.socket.TextMessage;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.handler.TextWebSocketHandler;

import javax.websocket.server.ServerEndpoint;

import java.io.IOException;
import java.net.InetAddress;
import java.util.Collection;
import java.util.Date;
import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;


/**
 * @Author: ZVerify
 * @Description: websocket服务
 * @DateTime: 2022/9/6 14:03
 **/
@Data
@Service
@ServerEndpoint(value = "/ws-connect")
public class ZVerifyWebSocketHandler extends TextWebSocketHandler {


    private static ChatRecordService chatRecordService;

    @Autowired
    public void setChatRecordDao(ChatRecordService chatRecordService) {
        ZVerifyWebSocketHandler.chatRecordService = chatRecordService;
    }

    private static UserInfoService userInfoService;

    @Autowired
    public void setUserInfoService(UserInfoService userInfoService) {
        ZVerifyWebSocketHandler.userInfoService = userInfoService;
    }

    private static UploadStrategyContext uploadStrategyContext;

    @Autowired
    public void setUploadStrategyContext(UploadStrategyContext uploadStrategyContext) {
        ZVerifyWebSocketHandler.uploadStrategyContext = uploadStrategyContext;
    }
    /**
     * 固定前缀
     */
    public static String HEADER_NAME = "X-Real-IP";

    /**
     * 存放Session集合，方便推送消息
     */
    private static ConcurrentHashMap<String, WebSocketSession> webSocketSessionMaps = new ConcurrentHashMap<>();



    // 监听：连接开启
    @Override
    public void afterConnectionEstablished(WebSocketSession session) throws Exception {

        // put到集合，方便后续操作
        String userId = session.getAttributes().get("userId").toString();
        webSocketSessionMaps.put(HEADER_NAME + userId, session);
        // 更新在线人数
        updateOnlineCount();

        // 加载历史聊天记录
        ChatRecordDTO chatRecordDTO = listChartRecords(session);

        // 发送消息
        WebsocketMessageDTO messageDTO = WebsocketMessageDTO.builder()
                .type(ChatTypeEnum.HISTORY_RECORD.getType())
                .data(chatRecordDTO)
                .build();
        synchronized (session) {
            session.sendMessage(new TextMessage(JSON.toJSONString(messageDTO)));
        }
        // 给个提示
        String tips = "Web-Socket 连接成功，sid=" + session.getId() + "，userId=" + userId;
        System.out.println(tips);

    }

    /**
     * 加载历史聊天记录
     *
     * @param session session
     * @return 加载历史聊天记录
     */
    private ChatRecordDTO listChartRecords(WebSocketSession session) {

        String ipAddress = session.getAcceptedProtocol();

        LambdaQueryWrapper<ChatRecord> queryWrapper = new LambdaQueryWrapper<>();

        queryWrapper.ge(ChatRecord::getCreateTime, DateUtil.offsetHour(new Date(), -12));

        return ChatRecordDTO.builder()
                .chatRecordList(chatRecordService.list(queryWrapper))
                .ipAddress(ipAddress)
                .ipSource(IpUtil.getIpSource(ipAddress))
                .build();
    }

    private void updateOnlineCount() throws IOException {

        // 获取当前在线人数
        WebsocketMessageDTO messageDTO = WebsocketMessageDTO.builder()
                .type(ChatTypeEnum.ONLINE_COUNT.getType())
                .data(webSocketSessionMaps.size())
                .build();
        // 广播消息
        broadcastMessage(messageDTO);
    }

    // 监听：连接关闭
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status){
        // 从集合移除
        String userId = session.getAttributes().get("userId").toString();
        webSocketSessionMaps.remove(HEADER_NAME + userId);

    }

    // 收到消息
    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) throws IOException {

        String ipAddress = null;
        WebsocketMessageDTO messageDTO = JSONUtil.toBean(message.getPayload(), WebsocketMessageDTO.class, false);
        switch (Objects.requireNonNull(ChatTypeEnum.getChatType(messageDTO.getType()))) {
            case SEND_MESSAGE:

                String data = String.valueOf(messageDTO.getData()) ;
                InetAddress address = Objects.requireNonNull(session.getLocalAddress()).getAddress();
                if (Opp.of(address).isNonNull()){

                    ipAddress = address.getHostAddress();
                }


                String userId = session.getAttributes().get("userId").toString();
                UserInfo byId = userInfoService.getById(Integer.valueOf(userId));

                // 发送消息
                ChatRecord chatRecord = new ChatRecord();

                chatRecord.setContent(data);
                chatRecord.setType(messageDTO.getType());
                chatRecord.setAvatar(byId.getAvatar());
                chatRecord.setNickname(byId.getNickname());
                chatRecord.setUserId(byId.getId());
                chatRecord.setIpAddress(ipAddress);
                String ipSource = IpUtil.getIpSource(ipAddress);
                chatRecord.setIpSource(ipSource);
                chatRecordService.save(chatRecord);

                messageDTO.setData(chatRecord);
                // 广播消息
                broadcastMessage(messageDTO);
                break;
            case RECALL_MESSAGE:
                // 撤回消息
                RecallMessageDTO recallMessage = JSON.parseObject(JSON.toJSONString(messageDTO.getData()), RecallMessageDTO.class);
                // 删除记录
                chatRecordService.removeById(recallMessage.getId());
                // 广播消息
                broadcastMessage(messageDTO);
                break;
            case HEART_BEAT:
                // 心跳消息
                messageDTO.setData("pong");
                session.sendMessage(new TextMessage((JSON.toJSONString(messageDTO))));

            default:
                break;
        }
    }

    // -----------

    // 向指定客户端推送消息
    public static void sendMessage(WebSocketSession session, String message) {
        try {
            System.out.println("向sid为：" + session.getId() + "，发送：" + message);
            session.sendMessage(new TextMessage(message));
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    // 向指定用户推送消息
    public static void sendMessage(long userId, String message) {
        WebSocketSession session = webSocketSessionMaps.get(HEADER_NAME + userId);
        if(session != null) {
            sendMessage(session, message);
        }
    }

    /**
     * 广播消息
     *
     * @param messageDTO 消息dto
     * @throws IOException io异常
     */
    private void broadcastMessage(WebsocketMessageDTO messageDTO) throws IOException {

        Collection<WebSocketSession> sessions = webSocketSessionMaps.values();

        for (WebSocketSession webSocketService : sessions) {
            synchronized (webSocketService){
                TextMessage textMessage = new TextMessage(JSON.toJSONString(messageDTO));
                webSocketService.sendMessage(textMessage);
            }

        }
    }

    /**
     * 发送语音
     *
     * @param voiceRO 语音路径
     */
    public void sendVoice(VoiceRO voiceRO) {
        // 上传语音文件
        String content = uploadStrategyContext.executeUploadStrategy(voiceRO.getFile(), FilePathEnum.VOICE.getPath());
        voiceRO.setContent(content);
        // 保存记录
        ChatRecord chatRecord = BeanCopyUtils.copyObject(voiceRO, ChatRecord.class);
        chatRecordService.save(chatRecord);
        // 发送消息
        WebsocketMessageDTO messageDTO = WebsocketMessageDTO.builder()
                .type(ChatTypeEnum.VOICE_MESSAGE.getType())
                .data(chatRecord)
                .build();
        // 广播消息
        try {
            broadcastMessage(messageDTO);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }



}
```