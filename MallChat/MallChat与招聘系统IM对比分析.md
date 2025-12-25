# MallChat vs 招聘系统 IM 深度对比

> 两个企业级 IM 系统的架构、设计和实现对比分析

---

## 📊 项目概览对比

| 维度 | MallChat（抹茶） | 招聘系统 IM |
|------|-----------------|------------|
| **业务场景** | 电商 + 即时通讯 | 招聘场景下的沟通 |
| **用户类型** | 普通用户 | 求职者 + 招聘者 |
| **聊天类型** | 群聊 + 单聊 | 职位聊天室（1对1） |
| **核心特性** | 热门群聊、消息互动、敏感词过滤 | 聊天室管理、标签系统、AI 自动回复 |
| **技术栈** | Netty + WebSocket | Spring WebSocket |
| **分布式** | RocketMQ | Redis Pub/Sub / RabbitMQ |
| **数据库** | MyBatis-Plus | MyBatis-Plus |
| **缓存** | Redis + Caffeine | Redis |

---

## 🏗️ 架构对比

### 1. WebSocket 实现方式

#### MallChat：Netty 原生实现

```java
// 优势：性能更高、更灵活
public class NettyWebSocketServer {
    private EventLoopGroup bossGroup = new NioEventLoopGroup(1);
    private EventLoopGroup workerGroup = new NioEventLoopGroup(NettyRuntime.availableProcessors());
    
    public void run() {
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) {
                    ChannelPipeline pipeline = ch.pipeline();
                    pipeline.addLast(new IdleStateHandler(30, 0, 0));
                    pipeline.addLast(new HttpServerCodec());
                    pipeline.addLast(new WebSocketServerProtocolHandler("/"));
                    pipeline.addLast(NETTY_WEB_SOCKET_SERVER_HANDLER);
                }
            });
        serverBootstrap.bind(8090).sync();
    }
}
```


**特点：**
- 手动管理线程池（Boss + Worker）
- 自定义 Handler 链路
- 更底层的控制能力
- 性能更优（适合高并发场景）

#### 招聘系统：Spring WebSocket

```java
// 优势：开发更简单、与 Spring 集成更好
@Configuration
@EnableWebSocket
public class WebSocketHandlerConfiguration implements WebSocketConfigurer {
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(zpJobChatWebSocketHandler(), "/ws/zp/job/chat")
            .addInterceptors(webSocketSessionInterceptor())
            .setAllowedOrigins("*");
    }
}

public class ZpJobChatWebSocketHandler extends AbstractWebSocketHandler {
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        // 处理消息
    }
}
```

**特点：**
- Spring 自动管理连接
- 配置简单，开箱即用
- 与 Spring 生态集成好
- 适合中小型项目

---

### 2. 连接管理对比

#### MallChat：三层 Map 结构

```java
// 1. 等待登录的连接（本地缓存）
Cache<Integer, Channel> WAIT_LOGIN_MAP

// 2. 所有在线的 WebSocket 连接
ConcurrentHashMap<Channel, WSChannelExtraDTO> ONLINE_WS_MAP

// 3. 用户ID与Channel的映射（支持多端登录）
ConcurrentHashMap<Long, CopyOnWriteArrayList<Channel>> ONLINE_UID_MAP
```

**特点：**
- 使用 Caffeine 本地缓存（登录码）
- 支持多端登录（一个用户多个 Channel）
- 直接操作 Netty Channel

#### 招聘系统：双层 Map 结构

```java
// 1. sessionId → WebSocketSession
Map<String, WebSocketSession> sessionMap

// 2. userId → Map<sessionId, UserSession>
Map<String, Map<String, UserSession>> userSessionMap
```

**特点：**
- 使用 Spring WebSocketSession
- 支持多端登录
- 更面向对象的设计

---


## 🔄 消息流程对比

### MallChat 消息流程

```
用户发送消息
  ↓
NettyWebSocketServerHandler.channelRead0()
  ↓
WebSocketService.handleLoginReq() / 其他处理
  ↓
ChatController.sendMsg()（HTTP 接口）
  ↓
ChatService.sendMsg()
  ↓
保存消息到数据库
  ↓
发送到 RocketMQ
  ↓
MsgSendConsumer 消费消息
  ↓
更新房间/会话时间
  ↓
PushService.sendPushMsg()
  ↓
WebSocketService.sendToUid()
  ↓
通过 Channel 推送给客户端
```

**特点：**
- WebSocket 主要用于接收消息和推送
- 发送消息通过 HTTP 接口
- 使用 RocketMQ 解耦
- 读扩散 + 写扩散混合模式

### 招聘系统消息流程

```
用户发送消息
  ↓
ZpJobChatWebSocketHandler.handleTextMessage()
  ↓
ZpJobChatWebSocketService.parseUserMessage()
  ↓
ZpJobChatWebSocketService.processUserMessage()
  ↓
ZpMessageFactory.getOperate()（策略模式）
  ↓
ZpJobChatMessageOperate.doHandleMessageOperate()
  ├─ beforeHandle()：创建/切换聊天室
  ├─ operateHandle()：保存消息、更新标签
  └─ afterHandle()：AI 自动回复
  ↓
SpringUtils.publishEvent(ZpWsMessageSendEvent)
  ↓
ZpWsMessageSendEventListener.onChatSend()
  ↓
ZpJobChatWebSocketService.publish()
  ↓
Redis Pub/Sub 或 RabbitMQ
  ↓
AbstractTopicService.handle()
  ↓
ServletWebSocketManager.sendMessage()
  ↓
推送给客户端
```

**特点：**
- WebSocket 用于收发消息
- 使用策略模式处理不同业务
- 使用 Spring Event 解耦
- 使用 Redis/RabbitMQ 分布式通信

---


## 🎨 设计模式对比

### MallChat

| 设计模式 | 应用场景 | 示例 |
|---------|---------|------|
| **工厂模式** | 创建不同类型的消息处理器 | MessageHandlerFactory |
| **策略模式** | 处理不同类型的消息内容 | TextMsgHandler, ImgMsgHandler |
| **适配器模式** | 适配不同的数据格式 | WSAdapter |
| **模板方法** | 定义消息处理流程 | AbstractMsgHandler |
| **观察者模式** | 事件驱动 | Spring Event |
| **单例模式** | 全局唯一的服务 | NettyWebSocketServer |

### 招聘系统

| 设计模式 | 应用场景 | 示例 |
|---------|---------|------|
| **策略模式** | 处理不同业务类型的消息 | ZpJobChatMessageOperate |
| **工厂模式** | 管理策略实例 | ZpMessageFactory |
| **模板方法** | 定义处理流程（before/operate/after） | ZpMessageOperate |
| **发布订阅** | 分布式消息通信 | TopicManager |
| **观察者模式** | 事件驱动 | Spring Event |
| **代理模式** | 事务管理 | getInstance() |

**对比：**
- MallChat 更注重消息类型的处理
- 招聘系统更注重业务流程的管理
- 招聘系统的模板方法模式更完善（三阶段）

---

## 💾 数据库设计对比

### MallChat

```sql
-- 消息表
message
├─ id：消息ID
├─ room_id：房间ID
├─ from_uid：发送者
├─ content：消息内容
├─ reply_msg_id：回复的消息
├─ type：消息类型
├─ extra：扩展字段（JSON）
└─ status：消息状态

-- 房间表
room
├─ id：房间ID
├─ type：房间类型（群聊/单聊）
├─ hot_flag：是否热门群聊
├─ last_msg_id：最后一条消息
└─ active_time：最后活跃时间

-- 会话表（用户维度）
contact
├─ id：会话ID
├─ uid：用户ID
├─ room_id：房间ID
├─ read_time：已读时间
└─ active_time：会话更新时间

-- 群成员表
group_member
├─ id：成员ID
├─ group_id：群组ID
└─ uid：用户ID
```


### 招聘系统

```sql
-- 聊天室表
chat_session
├─ id：聊天室ID
├─ biz_type：业务类型（ZP_JOB_CHAT）
├─ biz_id：业务ID（职位ID）
├─ source_type：来源类型（SEEK：求职者）
└─ source_id：来源ID（求职者ID）

-- 聊天室成员表
chat_room_member
├─ id：成员ID
├─ chat_session_id：聊天室ID
├─ entity_type：实体类型（USER/ACCOUNT）
├─ entity_id：实体ID
└─ room_show_ind：是否显示

-- 消息表
chat_message
├─ id：消息ID
├─ sender_entity_type：发送者类型
├─ sender_entity_id：发送者ID
├─ recipient_entity_type：接收者类型
├─ recipient_entity_id：接收者ID
├─ message_content_type：消息内容类型
└─ status：消息状态

-- 消息内容表（分离设计）
chat_message_content
├─ id：内容ID
├─ chat_session_id：聊天室ID
├─ chat_message_id：消息ID
└─ content：消息内容

-- 标签关系表（使用字典表）
dict_relation
├─ relation_biz_type：关系业务类型
├─ biz_type：业务类型
├─ biz_id：业务ID（聊天室ID）
└─ dict_ids：标签ID列表
```

**对比：**

| 维度 | MallChat | 招聘系统 |
|------|----------|---------|
| **消息存储** | 单表（message） | 双表（chat_message + chat_message_content） |
| **房间设计** | room 表 | chat_session 表（更业务化） |
| **成员管理** | group_member | chat_room_member（支持多种实体类型） |
| **标签系统** | 无 | 使用字典表 + 关系表 |
| **扩展性** | extra 字段（JSON） | 分表设计 |

**优劣分析：**
- MallChat：简单直接，适合通用 IM
- 招聘系统：更业务化，支持复杂场景

---


## 🚀 核心特性对比

### 1. 消息推送机制

#### MallChat：读扩散 + 写扩散混合

```java
// 热门群聊（写扩散）
if (room.isHotRoom()) {
    // 只更新 Room 表
    hotRoomCache.refreshActiveTime(room.getId(), message.getCreateTime());
    // 推送给所有在线用户
    pushService.sendPushMsg(WSAdapter.buildMsgSend(msgResp));
}

// 普通群聊/单聊（读扩散）
else {
    // 更新每个成员的 Contact 表
    contactDao.refreshOrCreateActiveTime(room.getId(), memberUidList, ...);
    // 只推送给房间成员
    pushService.sendPushMsg(WSAdapter.buildMsgSend(msgResp), memberUidList);
}
```

**优势：**
- 热门群聊减少数据库写入
- 普通群聊保证消息可靠性
- 根据场景选择策略

#### 招聘系统：纯读扩散

```java
// 所有消息都保存到数据库
chatMessageService.save(chatMessage);
chatMessageContentService.save(chatMessageContent);

// 推送给接收者
message.setReceivers(List.of(receiverId));
this.publish(message);
```

**优势：**
- 实现简单
- 消息可靠性高
- 适合 1 对 1 场景

---

### 2. 登录认证

#### MallChat：微信扫码登录

```java
// 1. 生成登录码
Integer code = generateLoginCode(channel);

// 2. 请求微信接口获取二维码
WxMpQrCodeTicket ticket = wxMpService.getQrcodeService()
    .qrCodeCreateTmpTicket(code, (int) EXPIRE_TIME.getSeconds());

// 3. 返回二维码给前端
sendMsg(channel, WSAdapter.buildLoginResp(ticket));

// 4. 用户扫码后回调
public Boolean scanLoginSuccess(Integer loginCode, Long uid) {
    Channel channel = WAIT_LOGIN_MAP.getIfPresent(loginCode);
    String token = loginService.login(uid);
    loginSuccess(channel, user, token);
}
```

**特点：**
- 无需密码
- 安全性高
- 用户体验好

#### 招聘系统：Token 认证

```java
// 1. 拦截器验证 Token
public class WebSocketSessionInterceptor extends HandshakeInterceptor {
    @Override
    public boolean beforeHandshake(ServerHttpRequest request, ...) {
        String token = getTokenFromRequest(request);
        if (loginService.verify(token)) {
            Long uid = loginService.getValidUid(token);
            attributes.put("uid", uid);
            return true;
        }
        return false;
    }
}

// 2. 连接建立后自动登录
public void afterConnectionEstablished(WebSocketSession session) {
    Long uid = (Long) session.getAttributes().get("uid");
    this.online(session, uid);
}
```

**特点：**
- 与现有登录系统集成
- 实现简单
- 适合企业内部系统

---


### 3. 分布式通信

#### MallChat：RocketMQ

```java
// 发送消息到 MQ
@RocketMQMessageListener(
    consumerGroup = MQConstant.SEND_MSG_GROUP, 
    topic = MQConstant.SEND_MSG_TOPIC
)
public class MsgSendConsumer implements RocketMQListener<MsgSendMessageDTO> {
    @Override
    public void onMessage(MsgSendMessageDTO dto) {
        // 处理消息
        Message message = messageDao.getById(dto.getMsgId());
        // 推送给用户
        pushService.sendPushMsg(msgResp, memberUidList);
    }
}
```

**优势：**
- 消息可靠性高（持久化）
- 支持消息重试
- 支持延迟消息
- 适合大型项目

**劣势：**
- 需要额外部署 RocketMQ
- 配置复杂

#### 招聘系统：Redis Pub/Sub 或 RabbitMQ

```java
// 支持两种方式
public void publish(T message) throws Exception {
    if (BroadcastType.Redisson.equals(this.manager.getBroadcastType())) {
        // Redis Pub/Sub
        this.manager.getRedissonUtils()
            .getTopic(config.getTopicName())
            .publish(message);
    } else if (BroadcastType.Rabbit.equals(this.manager.getBroadcastType())) {
        // RabbitMQ
        this.manager.getRabbitUtils()
            .send(config.getExchange(), config.getRoutingKey(), message);
    }
}
```

**优势：**
- 灵活选择（Redis 或 RabbitMQ）
- Redis 轻量级，适合中小型项目
- RabbitMQ 可靠性高，适合大型项目

**劣势：**
- Redis Pub/Sub 不持久化（消息可能丢失）

---

### 4. 特色功能

#### MallChat 特色

**1. 热门群聊**
```java
// 全员展示的群聊，使用写扩散
if (room.isHotRoom()) {
    // 推送给所有在线用户
    pushService.sendPushMsg(WSAdapter.buildMsgSend(msgResp));
}
```

**2. 消息互动**
- 点赞
- 撤回
- 删除
- 回复

**3. 敏感词过滤**
- DFA 算法
- AC 自动机

**4. IP 归属地解析**
```java
// 解析用户 IP 归属地
String ip = NettyUtil.getAttr(channel, NettyUtil.IP);
user.refreshIp(ip);
```

#### 招聘系统特色

**1. 聊天室管理**
```java
// 创建聊天室
ChatSessionVo room = chatSessionService.addRoom(
    "", 
    ZpChatSessionBizTypeEnum.ZP_JOB_CHAT.getValue(), 
    jobId,
    ZpChatSourceTypeEnum.SEEK.getValue(), 
    accountId
);

// 切换职位（切换聊天室）
if (!Objects.equals(room.getBizId(), message.getBizId())) {
    chatSessionService.changeRoomBiz(roomId, bizType, newJobId);
}
```

**2. 标签系统**
```java
// 打招呼 → 沟通中 → 合适/不合适
if (chatMessageService.checkCommunicating(chatSessionId)) {
    newDictIds.remove(greetingDict.getId());
    newDictIds.add(communicatingDict.getId());
}
```

**3. AI 自动回复**
```java
// afterHandle() 阶段触发
String response = aiFactory.getChatService()
    .chatCompletionText(request);

// 保存 AI 回复
ZpJobChatWebSocketMessage aiMessage = ...;
this.saveChatMessage(aiMessage, ChatMessageStatusEnum.SENT.getValue());
SpringUtils.publishEvent(new ZpWsMessageSendEvent(aiMessage));
```

**4. 消息状态管理**
- 已发送（SENT）
- 已送达（DELIVERED）
- 已读（READ）
- 失败（FAIL）

---


## 🔧 事务管理对比

### MallChat：简单事务

```java
@Transactional(rollbackFor = Exception.class)
public Long sendMsg(ChatMessageReq request, Long uid) {
    // 1. 保存消息
    Message message = messageDao.save(...);
    
    // 2. 发送到 MQ
    mqProducer.sendMsg(MQConstant.SEND_MSG_TOPIC, 
        new MsgSendMessageDTO(message.getId()));
    
    return message.getId();
}
```

**特点：**
- 单一事务
- 简单直接
- MQ 在事务内发送

### 招聘系统：多级事务

```java
// 主事务
@Transactional(rollbackFor = Exception.class)
protected void operateHandle(ZpJobChatWebSocketMessage message) {
    try {
        // 业务逻辑...
        
        // 保存消息（独立事务）
        this.getInstance().saveChatMessage(message, ChatMessageStatusEnum.SENT.getValue());
        
        // 发布事件
        SpringUtils.publishEvent(new ZpWsMessageSendEvent(message));
        
    } catch (Exception e) {
        // 失败：保存失败状态（独立事务）
        this.getInstance().saveChatMessage(message, ChatMessageStatusEnum.FAIL.getValue());
        SpringUtils.publishEvent(new ZpWsMessageSendEvent(message));
        throw new RuntimeException(e);
    }
}

// 独立事务
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
protected void saveChatMessage(ZpJobChatWebSocketMessage message, Integer messageStatus) {
    // 保存消息...
}

// 事务感知事件
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
public void onChatSend(ZpWsMessageSendEvent event) {
    // 事务提交后触发
    zpJobChatWebSocketService.publish(event.getMessage());
}
```

**特点：**
- 多级事务（REQUIRES_NEW）
- 事务感知事件（AFTER_COMPLETION）
- 即使主事务回滚，消息状态也已保存
- 更可靠

**对比：**
- MallChat：简单，适合大多数场景
- 招聘系统：复杂，但更可靠

---

## 📈 性能对比

### MallChat

**优势：**
1. **Netty 性能更高**
   - 零拷贝
   - 直接内存
   - 高效的线程模型

2. **本地缓存 + Redis**
   - Caffeine 本地缓存（登录码）
   - Redis 缓存（用户信息、群组信息）
   - 减少数据库查询

3. **读写分离**
   - 热门群聊用写扩散
   - 普通群聊用读扩散

**性能指标：**
- 单机支持 1-2 万并发连接
- 3 台服务器支持 3-6 万并发连接

### 招聘系统

**优势：**
1. **Spring WebSocket 简单**
   - 开发效率高
   - 维护成本低

2. **Redis 缓存**
   - 用户信息缓存
   - 聊天室信息缓存

3. **异步处理**
   - Spring Event 异步执行
   - 提高响应速度

**性能指标：**
- 适合中小型项目
- 单机支持 5000-10000 并发连接

**对比：**
- MallChat：性能更高，适合大型项目
- 招聘系统：开发效率高，适合中小型项目

---


## 🎯 适用场景

### MallChat 适合

✅ **大型社交/电商平台**
- 用户量大（百万级）
- 并发高（万级）
- 需要高性能

✅ **复杂的群聊场景**
- 热门群聊（全员展示）
- 普通群聊
- 单聊

✅ **需要消息互动**
- 点赞
- 撤回
- 回复

✅ **需要内容审核**
- 敏感词过滤
- 内容安全

### 招聘系统 适合

✅ **企业内部系统**
- 用户量中等（万级）
- 并发中等（千级）
- 业务场景明确

✅ **1 对 1 沟通场景**
- 客服系统
- 招聘系统
- 咨询系统

✅ **需要业务集成**
- 与现有系统集成
- 业务流程复杂
- 需要标签管理

✅ **需要 AI 能力**
- 自动回复
- 智能推荐
- 内容生成

---

## 📚 学习建议

### 如果你是初学者

**推荐学习顺序：**

1. **先学招聘系统**
   - Spring WebSocket 更简单
   - 与 Spring 生态集成好
   - 代码结构清晰
   - 适合快速上手

2. **再学 MallChat**
   - Netty 更底层
   - 性能优化更多
   - 适合深入学习

### 如果你要做项目

**选择建议：**

| 场景 | 推荐 | 理由 |
|------|------|------|
| 创业项目（MVP） | 招聘系统 | 开发快，成本低 |
| 企业内部系统 | 招聘系统 | 与 Spring 集成好 |
| 大型社交平台 | MallChat | 性能高，可扩展 |
| 电商平台 | MallChat | 支持复杂场景 |
| 客服系统 | 招聘系统 | 1 对 1 场景 |
| 直播聊天 | MallChat | 高并发 |

---

## 🔍 核心差异总结

### 技术选型

| 维度 | MallChat | 招聘系统 | 推荐 |
|------|----------|---------|------|
| **WebSocket** | Netty | Spring WebSocket | 看场景 |
| **消息队列** | RocketMQ | Redis/RabbitMQ | RocketMQ 更可靠 |
| **缓存** | Caffeine + Redis | Redis | Caffeine 性能更好 |
| **事务** | 简单事务 | 多级事务 | 多级事务更可靠 |

### 架构设计

| 维度 | MallChat | 招聘系统 | 推荐 |
|------|----------|---------|------|
| **分层** | 清晰 | 更清晰 | 招聘系统 |
| **设计模式** | 丰富 | 更丰富 | 招聘系统 |
| **扩展性** | 好 | 更好 | 招聘系统 |
| **复杂度** | 中等 | 较高 | 看场景 |

### 业务功能

| 功能 | MallChat | 招聘系统 | 说明 |
|------|----------|---------|------|
| **群聊** | ✅ | ❌ | MallChat 支持 |
| **单聊** | ✅ | ✅ | 都支持 |
| **聊天室** | ✅ | ✅ | 招聘系统更业务化 |
| **标签** | ❌ | ✅ | 招聘系统支持 |
| **AI** | ✅（机器人） | ✅（自动回复） | 招聘系统更智能 |
| **消息互动** | ✅ | ❌ | MallChat 支持 |
| **敏感词** | ✅ | ❌ | MallChat 支持 |

---


## 💡 面试回答模板

### Q: 你做过 IM 系统吗？说说你的实现方案

**回答思路：**

"我做过两个 IM 项目，一个是基于 Netty 的通用 IM 系统，一个是基于 Spring WebSocket 的招聘场景 IM。

**第一个项目（MallChat）：**
- 使用 Netty 实现 WebSocket 服务器，性能更高
- 支持群聊和单聊，热门群聊用写扩散，普通群聊用读扩散
- 使用 RocketMQ 解耦消息发送和推送
- 支持消息互动（点赞、撤回、回复）和敏感词过滤
- 单机支持 1-2 万并发连接

**第二个项目（招聘系统）：**
- 使用 Spring WebSocket，与 Spring 生态集成更好
- 采用分层架构，使用策略模式处理不同业务
- 使用 Spring Event 解耦业务逻辑
- 支持聊天室管理、标签系统、AI 自动回复
- 使用多级事务保证消息可靠性

两个项目各有优势，第一个性能更高，适合大型项目；第二个开发效率高，适合企业内部系统。"

---

### Q: Netty 和 Spring WebSocket 有什么区别？

**回答：**

"主要区别在于：

**Netty：**
- 更底层，性能更高（零拷贝、直接内存）
- 需要手动管理线程池和 Handler 链路
- 适合高并发场景（万级连接）
- 学习成本高

**Spring WebSocket：**
- 基于 Servlet 容器，与 Spring 集成好
- 配置简单，开箱即用
- 适合中小型项目（千级连接）
- 学习成本低

我在 MallChat 项目中用 Netty，因为需要支持万级并发；在招聘系统中用 Spring WebSocket，因为业务场景简单，开发效率更重要。"

---

### Q: 如何保证消息不丢失？

**回答：**

"我从多个层面保证消息可靠性：

**1. 数据库持久化**
- 消息先保存到数据库，再推送
- 即使推送失败，数据也不会丢

**2. 消息队列**
- 使用 RocketMQ 或 RabbitMQ 持久化消息
- 支持消息重试

**3. 消息状态管理**
- 已发送、已送达、已读、失败
- 失败的消息可以重新推送

**4. 多级事务**
- 使用 REQUIRES_NEW 独立事务保存消息状态
- 即使主事务回滚，消息状态也已保存

**5. 离线消息**
- 用户上线后，查询未读消息
- 重新推送

在招聘系统中，我还使用了事务感知事件（@TransactionalEventListener），确保消息在事务提交后才推送，避免推送成功但数据未保存的问题。"

---

### Q: 如何支持分布式部署？

**回答：**

"我使用发布订阅模式实现跨服务器通信：

**问题：**
- 用户A连接到服务器1
- 用户B连接到服务器2
- 用户A给用户B发消息，服务器1找不到用户B的连接

**解决方案：**
1. 服务器1收到消息，保存到数据库
2. 发布到 Redis 频道或 RabbitMQ 交换机
3. 所有服务器都订阅这个频道
4. 服务器2收到消息，推送给用户B

**实现细节：**
- MallChat 使用 RocketMQ，消息可靠性更高
- 招聘系统支持 Redis 和 RabbitMQ，可以根据项目需求选择
- Redis 轻量级，适合中小型项目
- RabbitMQ 可靠性高，适合大型项目

我还做了性能优化，只推送给相关用户，不是广播给所有服务器。"

---

### Q: 你们的 IM 系统有什么亮点？

**回答：**

"我总结了几个亮点：

**1. 读写分离策略（MallChat）**
- 热门群聊用写扩散，只更新 Room 表
- 普通群聊用读扩散，更新每个成员的 Contact 表
- 根据场景选择策略，性能更优

**2. 策略模式 + 模板方法（招聘系统）**
- 使用策略模式处理不同业务类型
- 使用模板方法定义处理流程（before/operate/after）
- 易于扩展，符合开闭原则

**3. 多级事务（招聘系统）**
- 使用 REQUIRES_NEW 独立事务保存消息状态
- 使用事务感知事件确保数据已提交
- 消息可靠性更高

**4. AI 自动回复（招聘系统）**
- 在 afterHandle() 阶段触发
- 不影响主流程
- 失败不影响用户消息

**5. 性能优化（MallChat）**
- Caffeine 本地缓存
- Redis 缓存
- 线程池异步处理
- 单机支持 1-2 万并发连接

这些设计都是根据实际业务场景总结出来的，不是为了设计而设计。"

---

## 🎓 总结

### 核心要点

1. **技术选型**
   - Netty：高性能，适合大型项目
   - Spring WebSocket：简单，适合中小型项目

2. **架构设计**
   - 分层清晰，职责单一
   - 使用设计模式（策略、工厂、模板方法）
   - 事件驱动，解耦业务逻辑

3. **消息可靠性**
   - 数据库持久化
   - 消息队列
   - 消息状态管理
   - 多级事务

4. **分布式支持**
   - 发布订阅模式
   - Redis Pub/Sub 或 RabbitMQ
   - 跨服务器通信

5. **性能优化**
   - 本地缓存 + Redis
   - 读写分离
   - 异步处理
   - 线程池管理

### 学习路径

```
1. 基础知识
   ├─ WebSocket 协议
   ├─ Netty 框架
   └─ Spring WebSocket

2. 架构设计
   ├─ 分层架构
   ├─ 设计模式
   └─ 事件驱动

3. 核心功能
   ├─ 连接管理
   ├─ 消息收发
   └─ 消息推送

4. 高级特性
   ├─ 分布式部署
   ├─ 消息可靠性
   └─ 性能优化

5. 实战项目
   ├─ 先学招聘系统（简单）
   └─ 再学 MallChat（复杂）
```

### 最后的话

两个项目都是优秀的 IM 实现，各有特点：

- **MallChat**：性能导向，适合学习高性能 IM 系统
- **招聘系统**：架构导向，适合学习企业级系统设计

建议两个都学习，对比理解，融会贯通。在实际项目中，根据业务场景选择合适的方案，不要盲目追求技术。

祝你学习顺利！💪
