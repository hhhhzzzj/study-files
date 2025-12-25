# 企业版 IM 系统深度解析

> 基于你已经掌握的 Version 1-5 知识，深入剖析企业版招聘聊天系统的架构设计

---

## 📋 目录

1. [系统概览](#系统概览)
2. [核心架构](#核心架构)
3. [分层设计](#分层设计)
4. [核心组件详解](#核心组件详解)
5. [消息流程](#消息流程)
6. [高级特性](#高级特性)
7. [与 Version 5 对比](#与-version-5-对比)
8. [面试要点](#面试要点)

---

## 🎯 系统概览

### 业务场景

这是一个**招聘场景下的 IM 系统**，核心特点：

```
参与者：
├─ 求职者（Account）：投递简历，与 HR 沟通
└─ 招聘者（User）：发布职位，筛选候选人

业务对象：
└─ 职位（Job）：聊天围绕具体职位展开

聊天室（ChatSession）：
├─ 一个职位 + 一个求职者 = 一个聊天室
├─ 支持切换职位（同一求职者，不同职位）
└─ 支持标签管理（打招呼、沟通中、合适、不合适）

特殊功能：
├─ AI 自动回复（招聘者可开启）
└─ 消息状态管理（已发送、已送达、已读、失败）
```

### 技术栈

```
后端框架：
├─ Spring Boot 3.2
├─ Spring WebSocket
├─ Spring Event
├─ Spring Transaction
└─ MyBatis-Plus

中间件：
├─ Redis（Redisson）：Pub/Sub 分布式消息
├─ RabbitMQ：消息队列（可选）
├─ MySQL：数据持久化
└─ AI 服务：智能回复

架构模式：
├─ 分层架构（Controller → Service → Repository）
├─ 策略模式（消息类型处理）
├─ 工厂模式（策略管理）
├─ 事件驱动（解耦业务逻辑）
├─ 发布订阅（分布式通信）
└─ 模板方法（before/operate/after）
```

---

## 🏗️ 核心架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ 求职者前端    │  │ 招聘者前端    │  │ 管理后台      │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │ WebSocket        │ WebSocket        │ HTTP
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         接入层                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ WebSocketHandlerConfiguration                            │  │
│  │  ├─ 路由配置：/ws/zp/job/chat                            │  │
│  │  ├─ 拦截器：WebSocketSessionInterceptor                  │  │
│  │  └─ 处理器：ZpJobChatWebSocketHandler                    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                         业务层                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ ZpJobChatWebSocketService                                │  │
│  │  ├─ 消息解析：parseUserMessage()                         │  │
│  │  ├─ 消息处理：processUserMessage()                       │  │
│  │  └─ 消息发布：publish()                                  │  │
│  └──────────────────────┬───────────────────────────────────┘  │
│                          │                                       │
│  ┌──────────────────────▼───────────────────────────────────┐  │
│  │ 策略模式：ZpMessageFactory                               │  │
│  │  ├─ ZpJobChatMessageOperate                              │  │
│  │  │   ├─ beforeHandle()：创建/切换聊天室                 │  │
│  │  │   ├─ operateHandle()：保存消息、更新标签             │  │
│  │  │   └─ afterHandle()：AI 自动回复                      │  │
│  │  └─ 其他策略...                                          │  │
│  └──────────────────────┬───────────────────────────────────┘  │
└─────────────────────────┼───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                         事件层                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Spring Event                                             │  │
│  │  ├─ 发布：SpringUtils.publishEvent()                    │  │
│  │  ├─ 事件：ZpWsMessageSendEvent                          │  │
│  │  └─ 监听：ZpWsMessageSendEventListener                  │  │
│  └──────────────────────┬───────────────────────────────────┘  │
└─────────────────────────┼───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                         分布式层                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ TopicManager（支持 Redis/RabbitMQ）                      │  │
│  │  ├─ 发布：publish()                                      │  │
│  │  ├─ 订阅：subscribe()                                    │  │
│  │  └─ 处理：handle()                                       │  │
│  └──────────────────────┬───────────────────────────────────┘  │
└─────────────────────────┼───────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
          ▼               ▼               ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Redis/RabbitMQ│  │ MySQL 数据库  │  │ AI 服务      │
└──────────────┘  └──────────────┘  └──────────────┘
```



---

## 📦 分层设计

### 1. 接入层（WebSocket Handler）

**职责：**
- 管理 WebSocket 连接生命周期
- 用户认证和会话管理
- 消息接收和分发

**核心类：**

```java
// 抽象基类：定义 WebSocket 生命周期
public abstract class AbstractWebSocketHandler extends TextWebSocketHandler {
    
    protected abstract ServletWebSocketManager getWebSocketManager();
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        // 1. 连接建立
        // 2. 创建会话
        this.getWebSocketManager().createSession(session);
    }
    
    @Override
    protected abstract void handleTextMessage(WebSocketSession session, TextMessage message);
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        // 1. 连接关闭
        // 2. 清理会话
        this.getWebSocketManager().closeSession(session);
    }
}

// 具体实现：招聘聊天处理器
public class ZpJobChatWebSocketHandler extends AbstractWebSocketHandler {
    
    private final ServletWebSocketManager servletWebSocketManager;
    private final ZpJobChatWebSocketService zpJobChatWebSocketService;
    
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        // 1. 检查用户登录状态
        if (!servletWebSocketManager.checkUserSession(session)) {
            return;
        }
        
        // 2. 解析用户消息
        JobChatUserMessage userMessage = zpJobChatWebSocketService.parseUserMessage(session, message);
        
        // 3. 处理业务逻辑
        zpJobChatWebSocketService.processUserMessage(session, userMessage);
    }
}
```

**关键点：**

1. **模板方法模式**：AbstractWebSocketHandler 定义流程，子类实现具体逻辑
2. **职责分离**：Handler 只负责连接管理，业务逻辑交给 Service
3. **统一异常处理**：handleTransportError() 统一处理传输错误

---

### 2. 会话管理层（ServletWebSocketManager）

**职责：**
- 管理所有 WebSocket 连接
- 支持多端登录（一个用户多个设备）
- 提供消息推送接口

**核心实现：**

```java
public class ServletWebSocketManager implements WebSocketManager<String, WebSocketSession> {
    
    // 双层 Map 结构
    private final Map<String, WebSocketSession> sessionMap = new ConcurrentHashMap<>();
    private final Map<String, Map<String, UserSession>> userSessionMap = new ConcurrentHashMap<>();
    
    // 创建会话
    public void createSession(WebSocketSession session) {
        String sessionId = session.getId();
        
        // 1. 保存 WebSocket 连接
        this.sessionMap.put(sessionId, session);
        
        // 2. 保存用户会话（支持多端登录）
        UserSession userSession = this.getUserSession(session);
        String userId = String.valueOf(userSession.getUid());
        
        userSessionMap.putIfAbsent(userId, new ConcurrentHashMap<>());
        userSessionMap.get(userId).put(sessionId, userSession);
    }
    
    // 获取用户的所有连接
    public List<WebSocketSession> getUserSessions(String userId) {
        return this.getUserSessionMap(userId).keySet().stream()
            .map(this.sessionMap::get)
            .toList();
    }
    
    // 发送消息（支持多种方式）
    public <M extends SimpleMessage<?>> void sendMessage(M message) {
        if (CollectionUtils.isNotEmpty(message.getKeys())) {
            // 方式1：根据 sessionId 发送
            message.getKeys().forEach(key -> {
                if (this.exist(key)) {
                    this.sendTextMessage(this.getSession(key), getMessageContent(message));
                }
            });
        } else if (CollectionUtils.isNotEmpty(message.getReceivers())) {
            // 方式2：根据 userId 发送（推送给所有设备）
            message.getReceivers().forEach(receiver -> {
                String key = String.valueOf(receiver);
                this.getUserSessions(key).forEach(s -> 
                    this.sendTextMessage(s, getMessageContent(message))
                );
            });
        } else {
            // 方式3：广播给所有连接
            this.getSessions().forEach(s -> 
                this.sendTextMessage(s, getMessageContent(message))
            );
        }
    }
}
```

**关键点：**

1. **双层 Map**：
   - 外层：userId → Map<sessionId, UserSession>
   - 内层：sessionId → UserSession
   - 支持一个用户多个设备

2. **线程安全**：使用 ConcurrentHashMap

3. **灵活推送**：
   - 按 sessionId 推送（单个连接）
   - 按 userId 推送（所有设备）
   - 广播（所有用户）

---

### 3. 业务服务层（ZpJobChatWebSocketService）

**职责：**
- 消息解析和验证
- 业务逻辑处理
- 消息发布到分布式层

**核心实现：**

```java
public class ZpJobChatWebSocketService extends ServletWebSocketService {
    
    private ImChatApi imChatApi;
    private ZpJobApi zpJobApi;
    
    // 解析用户消息
    public JobChatUserMessage parseUserMessage(WebSocketSession session, TextMessage textMessage) {
        // 1. JSON 解析
        JobChatUserMessage userMessage = JacksonUtils.toObject(textMessage.getPayload(), JobChatUserMessage.class);
        
        // 2. 获取用户会话
        UserSession userSession = this.getWebSocketManager().getUserSession(session);
        
        // 3. 判断用户类型（求职者/招聘者）
        ZpChatSourceTypeEnum sourceType = ZpChatSourceTypeEnum.getSourceType(userSession.getUsertype());
        userMessage.setSource(sourceType.getValue());
        
        // 4. 获取业务对象（职位信息）
        if (userMessage.getBizId() != null) {
            ZpJobChatVo bizObject = this.zpJobApi.getJobChatInfo(userMessage.getBizId());
            userMessage.setBizObject(bizObject);
        }
        
        // 5. 权限校验
        // - 求职者：应该是会话的发起人
        // - 招聘者：应该是职位的创建人/负责人/主管
        
        return userMessage;
    }
    
    // 处理用户消息
    public void processUserMessage(WebSocketSession session, JobChatUserMessage userMessage) {
        // 1. 根据消息类型分发
        if (ZpChatMessageContentTypeEnum.TEXT.getCode().equalsIgnoreCase(userMessage.getType())) {
            this.processTextMessage(session, userMessage);
        } else {
            // 其他类型...
        }
    }
    
    // 处理文本消息
    private void processTextMessage(WebSocketSession session, JobChatUserMessage userMessage) {
        UserSession userSession = this.getWebSocketManager().getUserSession(session);
        
        // 1. 保存消息到数据库
        ChatMessageSaveRequest request = ChatMessageSaveRequest.builder()
            .chatSessionId(userMessage.getChatSessionId())
            .type(ChatMessageContentTypeEnum.TEXT.getCode())
            .content(userMessage.getContent())
            .senderId(userSession.getUid())
            .receiverId(userMessage.getReceiverId())
            .build();
        this.imChatApi.saveChatMessage(request);
        
        // 2. 发布消息到分布式层
        SimpleTextMessage message = new SimpleTextMessage();
        message.setReceivers(List.of(userMessage.getReceiverId()));
        message.setContent(userMessage.getContent());
        this.publish(message);  // 继承自 ServletWebSocketService
    }
}
```

**关键点：**

1. **消息解析**：
   - JSON 反序列化
   - 用户身份识别
   - 业务对象加载
   - 权限校验

2. **业务处理**：
   - 保存到数据库
   - 发布到分布式层
   - 不直接推送（解耦）

3. **继承关系**：
   ```
   ServletWebSocketService
   └─ extends AbstractTopicService
       └─ implements TopicService
   ```

---

### 4. 策略层（ZpMessageOperate）

**职责：**
- 处理不同业务类型的消息
- 实现复杂的业务逻辑
- 支持事务管理

**核心实现：**

```java
// 抽象策略类（模板方法模式）
public abstract class ZpMessageOperate {
    
    protected abstract ZpWsMessageBizTypeEnum getBizTypeEnum();
    protected abstract ZpMessageOperate getInstance();  // 获取代理对象（支持事务）
    
    // 模板方法：定义处理流程
    public void doHandleMessageOperate(ZpJobChatWebSocketMessage message) {
        ZpMessageOperate operate = this.getInstance();
        try {
            operate.beforeHandle(message);   // 前置处理
            operate.operateHandle(message);  // 核心处理
            operate.afterHandle(message);    // 后置处理
        } catch (Exception e) {
            log.error("do handle message operate failed.", e);
        }
    }
    
    protected abstract void beforeHandle(ZpJobChatWebSocketMessage message);
    protected abstract void operateHandle(ZpJobChatWebSocketMessage message);
    protected abstract void afterHandle(ZpJobChatWebSocketMessage message);
}

// 具体策略：招聘聊天消息处理
@Component
public class ZpJobChatMessageOperate extends ZpMessageOperate {
    
    private final ChatSessionService chatSessionService;
    private final ChatMessageService chatMessageService;
    private final AiFactory aiFactory;
    
    @Override
    protected ZpWsMessageBizTypeEnum getBizTypeEnum() {
        return ZpWsMessageBizTypeEnum.ZP_JOB_CHAT;
    }
    
    // 前置处理：创建/切换聊天室
    @Transactional(rollbackFor = Exception.class)
    protected void beforeHandle(ZpJobChatWebSocketMessage message) {
        // 1. 如果没有聊天室，创建聊天室
        if (!ObjectUtils.isValidId(message.getChatSessionId())) {
            ChatSessionVo room = chatSessionService.addRoom(
                "", 
                ZpChatSessionBizTypeEnum.ZP_JOB_CHAT.getValue(), 
                message.getBizId(),
                ZpChatSourceTypeEnum.SEEK.getValue(), 
                entityId
            );
            message.setChatSessionId(room.getId());
            
            // 注册聊天室成员
            chatRoomMemberService.registerMember(room.getId(), message.getSenderEntityId(), message.getSenderEntityType());
            chatRoomMemberService.registerMember(room.getId(), message.getRecipientEntityId(), message.getRecipientEntityType());
        }
        
        // 2. 如果切换职位，更新聊天室
        ChatSessionEntity room = chatSessionService.findCacheById(message.getChatSessionId());
        if (!Objects.equals(room.getBizId(), message.getBizId())) {
            // 切换聊天室业务对象
            chatSessionService.changeRoomBiz(roomId, message.getBizType(), message.getBizId());
            // 移除旧成员，注册新成员
            zpChatRoomMemberService.unregisterMemberAll(roomId);
            chatRoomMemberService.registerMember(roomId, accountId, UserTypeEnum.ACCOUNT.getValue());
            chatRoomMemberService.registerMember(roomId, job.getCreatedBy(), UserTypeEnum.USER.getValue());
        }
    }
    
    // 核心处理：保存消息、更新标签
    @Transactional(rollbackFor = Exception.class)
    protected void operateHandle(ZpJobChatWebSocketMessage message) {
        try {
            // 1. 获取当前标签
            RelationVo<DictVo> relation = dictApi.getRelation(...);
            Set<Long> newDictIds = Sets.newHashSet();
            
            // 2. 标记"打招呼"状态
            DictVo greetingDict = dictApi.findByCode(ZpLabelBizTypeEnum.GREETING.getValue());
            newDictIds.add(greetingDict.getId());
            
            // 3. 如果已经沟通过，标记"沟通中"状态
            if (chatMessageService.checkCommunicating(message.getChatSessionId())) {
                newDictIds.remove(greetingDict.getId());
                DictVo communicatingDict = dictApi.findByCode(ZpLabelBizTypeEnum.COMMUNICATING.getValue());
                newDictIds.add(communicatingDict.getId());
            }
            
            // 4. 如果招聘者标记"不合适"
            if (ZpChatMessageContentTypeEnum.SQUARE_PEG.getValue().equalsIgnoreCase(message.getContentType())) {
                // 移除"合适"标签，添加"不合适"标签
                ...
            }
            
            // 5. 保存消息到数据库
            this.getInstance().saveChatMessage(message, ChatMessageStatusEnum.SENT.getValue());
            
            // 6. 更新标签
            if (!newDictIds.equals(oriDictIds)) {
                dictApi.saveRelation(...);
            }
            
            // 7. 显示隐藏的聊天室
            List<ChatRoomMemberEntity> members = chatRoomMemberService.listCanContactMemberByRoomId(...);
            // 如果有成员隐藏了聊天室，重新显示
            ...
            
            // 8. 发布事件（触发消息推送）
            SpringUtils.publishEvent(new ZpWsMessageSendEvent(message));
            
        } catch (Exception e) {
            // 失败：标记消息状态为失败
            this.getInstance().saveChatMessage(message, ChatMessageStatusEnum.FAIL.getValue());
            SpringUtils.publishEvent(new ZpWsMessageSendEvent(message));
            throw new RuntimeException(e);  // 触发事务回滚
        }
    }
    
    // 后置处理：AI 自动回复
    @Transactional(rollbackFor = Exception.class)
    protected void afterHandle(ZpJobChatWebSocketMessage message) {
        // 1. 检查是否开启 AI
        if (aiFactory == null || !UserTypeEnum.ACCOUNT.getCode().equalsIgnoreCase(message.getSenderEntityType())) {
            return;
        }
        
        ZpJobEntity job = zpJobService.findCacheById(message.getBizId());
        if (job == null || !BooleanTypeEnum.TRUE.getValue().equals(job.getAiInd())) {
            return;
        }
        
        // 2. 调用 AI 服务
        SimpleChatRequest request = SimpleChatRequest.builder()
            .conversationId(message.getAiConversationId())
            .prompt(message.getContent())
            .build();
        
        try {
            String response = aiFactory.getChatService().chatCompletionText(request);
            
            // 3. 构造 AI 回复消息
            ZpJobChatWebSocketMessage aiMessage = ZpJobChatWebSocketMessage.builder()
                .chatSessionId(message.getChatSessionId())
                .recipientEntityId(message.getSenderEntityId())
                .senderEntityId(message.getRecipientEntityId())
                .content(response)
                .ownerEntityType("ASSISTANT")
                .build();
            
            // 4. 保存 AI 消息
            this.getInstance().saveChatMessage(aiMessage, ChatMessageStatusEnum.SENT.getValue());
            
            // 5. 发布事件（推送 AI 回复）
            SpringUtils.publishEvent(new ZpWsMessageSendEvent(aiMessage));
            
        } catch (Exception e) {
            // AI 失败：保存失败状态
            ...
        }
    }
    
    // 保存消息（独立事务）
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    protected void saveChatMessage(ZpJobChatWebSocketMessage message, Integer messageStatus) {
        // 1. 保存消息主表
        ChatMessageEntity chatMessage = new ChatMessageEntity();
        chatMessage.setId(message.getMessageId());
        chatMessage.setSenderEntityType(message.getSenderEntityType());
        chatMessage.setSenderEntityId(message.getSenderEntityId());
        chatMessage.setRecipientEntityType(message.getRecipientEntityType());
        chatMessage.setRecipientEntityId(message.getRecipientEntityId());
        chatMessage.setMessageContentType(message.getContentType());
        chatMessage.setStatus(messageStatus);
        chatMessageService.save(chatMessage);
        
        // 2. 保存消息内容表
        ChatMessageContentEntity chatMessageContent = new ChatMessageContentEntity();
        chatMessageContent.setId(message.getMessageContentId());
        chatMessageContent.setChatSessionId(session.getId());
        chatMessageContent.setChatMessageId(chatMessage.getId());
        chatMessageContent.setContent(message.getContent());
        chatMessageContentService.save(chatMessageContent);
        
        message.setMessageId(chatMessage.getId());
        message.setMessageContentId(chatMessageContent.getId());
    }
}
```

**关键点：**

1. **模板方法模式**：
   - doHandleMessageOperate() 定义流程
   - before/operate/after 三个阶段
   - 子类实现具体逻辑

2. **事务管理**：
   - @Transactional：方法级事务
   - REQUIRES_NEW：独立事务（saveChatMessage）
   - 失败回滚，但消息状态已保存

3. **业务复杂度**：
   - 聊天室管理（创建、切换）
   - 标签管理（打招呼、沟通中、合适、不合适）
   - AI 自动回复
   - 消息状态管理

4. **getInstance()**：
   - 获取 Spring 代理对象
   - 支持事务、AOP
   - 避免 this 调用导致事务失效



---

### 5. 事件层（Spring Event）

**职责：**
- 解耦业务逻辑和消息推送
- 支持异步处理
- 支持事务感知

**核心实现：**

```java
// 事件定义
@Data
@AllArgsConstructor
public class ZpWsMessageSendEvent {
    private ZpJobChatWebSocketMessage message;
}

// 事件监听器
@Component
@RequiredArgsConstructor
public class ZpWsMessageSendEventListener {
    
    private final ZpJobChatWebSocketService zpJobChatWebSocketService;
    
    // 事务完成后触发（确保数据已提交）
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void onChatSend(ZpWsMessageSendEvent event) throws Exception {
        ZpJobChatWebSocketMessage message = event.getMessage();
        
        // 发布到分布式层（Redis/RabbitMQ）
        zpJobChatWebSocketService.publish(message);
    }
}

// 事件发布
SpringUtils.publishEvent(new ZpWsMessageSendEvent(message));
```

**关键点：**

1. **@TransactionalEventListener**：
   - AFTER_COMPLETION：事务提交后触发
   - 确保数据已保存到数据库
   - 避免消息推送成功但数据未保存

2. **解耦**：
   - 策略层只负责保存数据和发布事件
   - 监听器负责消息推送
   - 添加新功能只需要添加监听器

3. **异步支持**：
   - 可以配置 @Async 异步执行
   - 提高响应速度

---

### 6. 分布式层（TopicManager）

**职责：**
- 支持 Redis Pub/Sub 和 RabbitMQ
- 消息发布和订阅
- 跨服务器通信

**核心实现：**

```java
// TopicManager 接口
public interface TopicManager {
    Long getNodeId();                    // 节点ID
    BroadcastType getBroadcastType();    // 广播类型（Redis/RabbitMQ）
    RedissonUtils getRedissonUtils();    // Redis 工具
    RabbitUtils getRabbitUtils();        // RabbitMQ 工具
}

// AbstractTopicService：抽象主题服务
public abstract class AbstractTopicService<T extends SimpleMessage<?>, S> 
    implements TopicService<T> {
    
    private final TopicConfig config;
    private final TopicManager manager;
    private final WebSocketManager<String, S> webSocketManager;
    
    // 初始化：订阅主题
    private void init() {
        Class<T> messageType = GenericsUtils.getSuperGenericType(getClass(), AbstractTopicService.class, 0);
        
        if (BroadcastType.Redisson.equals(this.manager.getBroadcastType())) {
            // Redis Pub/Sub
            log.info("Init topic [{}] with redisson.", config.getName());
            manager.getRedissonUtils().addListener(config.getName(), messageType, (channel, msg) -> {
                log.info("Receive message [{}] from channel [{}].", msg, channel);
                this.handle(msg);  // 处理消息
            });
        } else if (BroadcastType.Rabbit.equals(this.manager.getBroadcastType())) {
            // RabbitMQ
            log.info("Init topic [{}] with rabbitmq.", config.getName());
            manager.getRabbitUtils().subscribeFanout(
                manager.getNodeId(), 
                config.getExchange(), 
                config.getDlx(), 
                messageType,
                (msg) -> {
                    log.info("Receive message [{}] from exchange [{}].", msg, config.getExchange());
                    this.handle(msg);  // 处理消息
                }, 
                () -> log.warn("RabbitMQ Subscriber failed.")
            );
        }
    }
    
    // 发布消息
    public void publish(T message) throws Exception {
        if (BroadcastType.Redisson.equals(this.manager.getBroadcastType())) {
            // 发布到 Redis
            log.info("Publish message [{}] to redis topic [{}].", message, config.getTopicName());
            this.manager.getRedissonUtils().getTopic(config.getTopicName()).publish(message);
        } else if (BroadcastType.Rabbit.equals(this.manager.getBroadcastType())) {
            // 发布到 RabbitMQ
            log.info("Publish message [{}] to rabbit topic [{}].", message, config.getTopicName());
            this.manager.getRabbitUtils().send(this.config.getExchange(), this.config.getRoutingKey(), message);
        }
    }
    
    // 处理消息（收到订阅消息时调用）
    @Override
    public void handle(T message) {
        log.info("Topic [{}] Receive message [{}]", config.getTopicName(), message);
        // 推送给本地的 WebSocket 连接
        this.webSocketManager.sendMessage(message);
    }
}

// ServletWebSocketService：Servlet WebSocket 服务
public class ServletWebSocketService extends AbstractTopicService<SimpleMessage<?>, WebSocketSession> {
    
    public ServletWebSocketService(TopicConfig config, 
                                   TopicManager topicManager, 
                                   ServletWebSocketManager servletWebSocketManager) {
        super(config, topicManager, servletWebSocketManager);
    }
}

// ZpJobChatWebSocketService：招聘聊天服务
public class ZpJobChatWebSocketService extends ServletWebSocketService {
    
    public ZpJobChatWebSocketService(TopicManager topicManager, 
                                     ServletWebSocketManager webSocketManager) {
        super(
            TopicConfig.builder().name("zp_job_chat_socket").build(), 
            topicManager, 
            webSocketManager
        );
        log.info("Create ZpJobChatWebSocketService with topic : {}", this.getConfig().getName());
    }
}
```

**关键点：**

1. **支持多种中间件**：
   - Redis Pub/Sub：轻量级，适合中小型项目
   - RabbitMQ：可靠性高，适合大型项目

2. **自动订阅**：
   - 服务启动时自动订阅主题
   - 收到消息自动推送给本地连接

3. **泛型设计**：
   - AbstractTopicService<T, S>
   - T：消息类型
   - S：会话类型（WebSocketSession）

4. **继承关系**：
   ```
   ZpJobChatWebSocketService
   └─ extends ServletWebSocketService
       └─ extends AbstractTopicService
           └─ implements TopicService
   ```

---

## 🔄 完整消息流程

### 场景：求职者给招聘者发消息

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 客户端发送消息                                                │
├─────────────────────────────────────────────────────────────────┤
│ WebSocket 客户端：                                               │
│ {                                                                │
│   "type": "TEXT",                                                │
│   "chatSessionId": 12345,                                        │
│   "bizId": 67890,                                                │
│   "receiverId": 10001,                                           │
│   "content": "您好，我对这个职位很感兴趣"                         │
│ }                                                                │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 接入层：ZpJobChatWebSocketHandler                             │
├─────────────────────────────────────────────────────────────────┤
│ handleTextMessage():                                             │
│ ├─ 检查用户登录状态                                              │
│ ├─ 解析消息：parseUserMessage()                                  │
│ │   ├─ JSON 反序列化                                             │
│ │   ├─ 获取用户会话（UserSession）                               │
│ │   ├─ 判断用户类型（求职者/招聘者）                             │
│ │   ├─ 加载业务对象（职位信息）                                  │
│ │   └─ 权限校验                                                  │
│ └─ 处理消息：processUserMessage()                                │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 业务层：ZpJobChatWebSocketService                             │
├─────────────────────────────────────────────────────────────────┤
│ processTextMessage():                                            │
│ ├─ 保存消息到数据库（imChatApi.saveChatMessage）                 │
│ └─ 发布消息到分布式层（this.publish）                            │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 策略层：ZpJobChatMessageOperate                               │
├─────────────────────────────────────────────────────────────────┤
│ doHandleMessageOperate():                                        │
│ ├─ beforeHandle()：创建/切换聊天室                               │
│ │   ├─ 如果没有聊天室，创建聊天室                                │
│ │   ├─ 注册聊天室成员                                            │
│ │   └─ 如果切换职位，更新聊天室                                  │
│ ├─ operateHandle()：保存消息、更新标签                           │
│ │   ├─ 获取当前标签                                              │
│ │   ├─ 更新标签（打招呼 → 沟通中）                               │
│ │   ├─ 保存消息到数据库（chat_message + chat_message_content）   │
│ │   ├─ 更新聊天室显示状态                                        │
│ │   └─ 发布事件：SpringUtils.publishEvent(ZpWsMessageSendEvent)  │
│ └─ afterHandle()：AI 自动回复                                    │
│     ├─ 检查是否开启 AI                                           │
│     ├─ 调用 AI 服务                                              │
│     ├─ 保存 AI 回复消息                                          │
│     └─ 发布事件（推送 AI 回复）                                  │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 事件层：ZpWsMessageSendEventListener                          │
├─────────────────────────────────────────────────────────────────┤
│ @TransactionalEventListener(phase = AFTER_COMPLETION)            │
│ onChatSend():                                                    │
│ └─ 发布到分布式层：zpJobChatWebSocketService.publish(message)    │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 分布式层：AbstractTopicService                                │
├─────────────────────────────────────────────────────────────────┤
│ publish():                                                       │
│ ├─ 如果是 Redis：                                                │
│ │   └─ redissonUtils.getTopic("zp_job_chat_socket").publish()   │
│ └─ 如果是 RabbitMQ：                                             │
│     └─ rabbitUtils.send(exchange, routingKey, message)           │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. Redis/RabbitMQ                                                │
├─────────────────────────────────────────────────────────────────┤
│ 广播消息给所有订阅了 "zp_job_chat_socket" 的服务器               │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. 分布式层：AbstractTopicService（所有服务器）                  │
├─────────────────────────────────────────────────────────────────┤
│ handle():                                                        │
│ └─ webSocketManager.sendMessage(message)                         │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ 9. 会话管理层：ServletWebSocketManager                           │
├─────────────────────────────────────────────────────────────────┤
│ sendMessage():                                                   │
│ ├─ 根据 receiverId 获取所有连接                                  │
│ ├─ 遍历所有连接                                                  │
│ └─ 推送消息：session.sendMessage(new TextMessage(json))          │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ 10. 客户端收到消息                                               │
├─────────────────────────────────────────────────────────────────┤
│ WebSocket 客户端：                                               │
│ onmessage = (event) => {                                         │
│   const message = JSON.parse(event.data);                        │
│   console.log("收到消息：", message.content);                     │
│ }                                                                │
└─────────────────────────────────────────────────────────────────┘
```

**时序图：**

```
求职者客户端    Handler    Service    Strategy    Event    Redis    Manager    招聘者客户端
    │             │          │          │          │        │        │            │
    │─发送消息────>│          │          │          │        │        │            │
    │             │─解析────>│          │          │        │        │            │
    │             │          │─处理────>│          │        │        │            │
    │             │          │          │─before──>│        │        │            │
    │             │          │          │─operate─>│        │        │            │
    │             │          │          │  保存DB  │        │        │            │
    │             │          │          │─发布事件>│        │        │            │
    │             │          │          │          │─监听──>│        │            │
    │             │          │          │          │        │─发布─>│            │
    │             │          │          │          │        │<─广播─│            │
    │             │          │          │          │        │        │─推送──────>│
    │             │          │          │<─after───│        │        │            │
    │             │          │          │  AI回复  │        │        │            │
    │             │          │          │─发布事件>│        │        │            │
    │             │          │          │          │─监听──>│        │            │
    │             │          │          │          │        │─发布─>│            │
    │             │          │          │          │        │<─广播─│            │
    │             │          │          │          │        │        │─推送AI────>│
```

---

## 🎨 高级特性

### 1. 聊天室管理

**核心概念：**

```
聊天室（ChatSession）：
├─ 一个职位 + 一个求职者 = 一个聊天室
├─ 支持切换职位（同一求职者，不同职位）
└─ 支持标签管理

数据库设计：
chat_session（聊天室表）
├─ id：聊天室ID
├─ biz_type：业务类型（ZP_JOB_CHAT）
├─ biz_id：业务ID（职位ID）
├─ source_type：来源类型（SEEK：求职者发起）
└─ source_id：来源ID（求职者ID）

chat_room_member（聊天室成员表）
├─ id：成员ID
├─ chat_session_id：聊天室ID
├─ entity_type：实体类型（USER/ACCOUNT）
├─ entity_id：实体ID
└─ room_show_ind：是否显示（TRUE/FALSE）
```

**切换职位流程：**

```java
// 场景：求职者从职位A切换到职位B

// 1. 检测到职位切换
if (!Objects.equals(room.getBizId(), message.getBizId())) {
    // 2. 更新聊天室业务对象
    chatSessionService.changeRoomBiz(roomId, message.getBizType(), message.getBizId());
    
    // 3. 移除旧成员
    zpChatRoomMemberService.unregisterMemberAll(roomId);
    
    // 4. 注册新成员
    chatRoomMemberService.registerMember(roomId, accountId, UserTypeEnum.ACCOUNT.getValue());
    chatRoomMemberService.registerMember(roomId, newJob.getCreatedBy(), UserTypeEnum.USER.getValue());
}
```

---

### 2. 标签管理

**标签类型：**

```
ZpLabelBizTypeEnum：
├─ GREETING：打招呼（首次发消息）
├─ COMMUNICATING：沟通中（已经互相发过消息）
├─ SUITABLE：合适（招聘者标记）
└─ UNSUITABLE：不合适（招聘者标记）

数据库设计：
使用字典表（dict）+ 关系表（dict_relation）
dict_relation：
├─ relation_biz_type：关系业务类型（ZP_CHAT_LABEL）
├─ biz_type：业务类型（ZP_LABEL）
├─ biz_id：业务ID（聊天室ID）
└─ dict_ids：标签ID列表
```

**标签更新流程：**

```java
// 1. 获取当前标签
RelationVo<DictVo> relation = dictApi.getRelation(...);
Set<Long> newDictIds = Sets.newHashSet();

// 2. 首次发消息：添加"打招呼"标签
DictVo greetingDict = dictApi.findByCode(ZpLabelBizTypeEnum.GREETING.getValue());
newDictIds.add(greetingDict.getId());

// 3. 已经互相发过消息：移除"打招呼"，添加"沟通中"
if (chatMessageService.checkCommunicating(message.getChatSessionId())) {
    newDictIds.remove(greetingDict.getId());
    DictVo communicatingDict = dictApi.findByCode(ZpLabelBizTypeEnum.COMMUNICATING.getValue());
    newDictIds.add(communicatingDict.getId());
}

// 4. 招聘者标记"不合适"
if (ZpChatMessageContentTypeEnum.SQUARE_PEG.getValue().equalsIgnoreCase(message.getContentType())) {
    DictVo suitableDict = dictApi.findByCode(ZpLabelBizTypeEnum.SUITABLE.getValue());
    newDictIds.remove(suitableDict.getId());
    
    DictVo unsuitableDict = dictApi.findByCode(ZpLabelBizTypeEnum.UNSUITABLE.getValue());
    newDictIds.add(unsuitableDict.getId());
}

// 5. 保存标签
if (!newDictIds.equals(oriDictIds)) {
    dictApi.saveRelation(...);
}
```



---

### 3. AI 自动回复

**工作流程：**

```java
// afterHandle() 阶段触发

// 1. 检查条件
if (aiFactory == null) return;  // AI 服务未配置
if (!UserTypeEnum.ACCOUNT.getCode().equalsIgnoreCase(message.getSenderEntityType())) return;  // 只有求职者发消息才触发

ZpJobEntity job = zpJobService.findCacheById(message.getBizId());
if (job == null || !BooleanTypeEnum.TRUE.getValue().equals(job.getAiInd())) return;  // 职位未开启 AI

// 2. 调用 AI 服务
SimpleChatRequest request = SimpleChatRequest.builder()
    .conversationId(message.getAiConversationId())  // 会话ID（保持上下文）
    .prompt(message.getContent())                   // 用户消息
    .build();

String response = aiFactory.getChatService().chatCompletionText(request);

// 3. 构造 AI 回复消息
ZpJobChatWebSocketMessage aiMessage = ZpJobChatWebSocketMessage.builder()
    .chatSessionId(message.getChatSessionId())
    .recipientEntityId(message.getSenderEntityId())  // 发给求职者
    .senderEntityId(message.getRecipientEntityId())  // 来自招聘者
    .content(response)
    .ownerEntityType("ASSISTANT")  // 标记为 AI 回复
    .build();

// 4. 保存 AI 消息
this.getInstance().saveChatMessage(aiMessage, ChatMessageStatusEnum.SENT.getValue());

// 5. 发布事件（推送 AI 回复）
SpringUtils.publishEvent(new ZpWsMessageSendEvent(aiMessage));
```

**关键点：**

1. **会话上下文**：
   - 使用 conversationId 保持上下文
   - AI 能记住之前的对话

2. **异步处理**：
   - afterHandle() 在事务提交后执行
   - 不影响主流程

3. **失败处理**：
   - AI 失败不影响用户消息
   - 保存失败状态，方便排查

---

### 4. 消息状态管理

**状态类型：**

```java
public enum ChatMessageStatusEnum {
    SENT(1, "已发送"),      // 消息已保存到数据库
    DELIVERED(2, "已送达"),  // 消息已推送给接收者
    READ(3, "已读"),        // 接收者已读
    FAIL(4, "失败");        // 发送失败
}
```

**状态更新流程：**

```
1. 发送者发送消息
   └─ 保存到数据库：status = SENT

2. 推送给接收者
   └─ 接收者在线：status = DELIVERED
   └─ 接收者离线：status = SENT（等待上线）

3. 接收者阅读消息
   └─ 发送已读回执：status = READ

4. 发送失败
   └─ 保存失败状态：status = FAIL
```

---

### 5. 多端登录

**实现原理：**

```java
// 双层 Map 结构
private final Map<String, Map<String, UserSession>> userSessionMap = new ConcurrentHashMap<>();

// 外层 Key：userId
// 内层 Key：sessionId
// Value：UserSession

// 用户连接
public void createSession(WebSocketSession session) {
    UserSession userSession = this.getUserSession(session);
    String userId = String.valueOf(userSession.getUid());
    
    // 添加到内层 Map
    userSessionMap.putIfAbsent(userId, new ConcurrentHashMap<>());
    userSessionMap.get(userId).put(session.getId(), userSession);
}

// 推送消息给所有设备
public List<WebSocketSession> getUserSessions(String userId) {
    return this.getUserSessionMap(userId).keySet().stream()
        .map(this.sessionMap::get)
        .toList();
}
```

**场景：**

```
用户 user1 同时用手机和电脑登录：

userSessionMap = {
    "user1": {
        "session_001": UserSession(手机),
        "session_002": UserSession(电脑)
    }
}

发送消息给 user1：
1. 获取 user1 的所有 session
2. 遍历推送：
   - session_001.sendMessage(message)  // 手机收到
   - session_002.sendMessage(message)  // 电脑收到
```

---

### 6. 事务管理

**事务传播：**

```java
// 主事务：operateHandle()
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
        throw new RuntimeException(e);  // 触发主事务回滚
    }
}

// 独立事务：saveChatMessage()
@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
protected void saveChatMessage(ZpJobChatWebSocketMessage message, Integer messageStatus) {
    // 保存消息...
}
```

**关键点：**

1. **REQUIRES_NEW**：
   - 创建新事务
   - 不受主事务影响
   - 即使主事务回滚，消息状态也已保存

2. **getInstance()**：
   - 获取 Spring 代理对象
   - 支持事务、AOP
   - 避免 this 调用导致事务失效

3. **事务感知事件**：
   - @TransactionalEventListener(phase = AFTER_COMPLETION)
   - 事务提交后触发
   - 确保数据已保存

---

## 🆚 与 Version 5 对比

### 架构对比

| 层次 | Version 5 | 企业版 | 说明 |
|------|-----------|--------|------|
| **接入层** | SimpleChatRoomV5 | ZpJobChatWebSocketHandler | 企业版有抽象基类 |
| **会话管理** | SessionManager | ServletWebSocketManager | 企业版支持更多功能 |
| **业务层** | 直接在 Handler | ZpJobChatWebSocketService | 企业版分层更清晰 |
| **策略层** | MessageHandler | ZpMessageOperate | 企业版用模板方法 |
| **事件层** | MessageSentEvent | ZpWsMessageSendEvent | 企业版支持事务感知 |
| **分布式层** | RedisMessageService | AbstractTopicService | 企业版支持多种中间件 |

### 功能对比

| 功能 | Version 5 | 企业版 | 差异 |
|------|-----------|--------|------|
| **消息类型** | 文本、图片、文件 | 文本、图片、文件、系统消息 | 企业版更丰富 |
| **聊天室** | 无 | 有（创建、切换、成员管理） | 企业版支持聊天室 |
| **标签** | 无 | 有（打招呼、沟通中、合适、不合适） | 企业版支持标签 |
| **AI** | 无 | 有（自动回复） | 企业版支持 AI |
| **消息状态** | 简单 | 完整（已发送、已送达、已读、失败） | 企业版更完善 |
| **事务** | 无 | 有（多级事务、事务感知事件） | 企业版更可靠 |
| **数据库** | 单表 | 多表（消息表、内容表、聊天室表、成员表） | 企业版更规范 |

### 代码对比

#### 1. 消息处理

**Version 5：**

```java
public void handle(ChatMessage message) {
    // 1. 验证
    validate(message);
    
    // 2. 保存数据库
    repository.save(message);
    
    // 3. 发布事件
    eventPublisher.publishEvent(new MessageSentEvent(this, message));
}
```

**企业版：**

```java
public void doHandleMessageOperate(ZpJobChatWebSocketMessage message) {
    ZpMessageOperate operate = this.getInstance();
    try {
        operate.beforeHandle(message);   // 前置：创建聊天室
        operate.operateHandle(message);  // 核心：保存消息、更新标签
        operate.afterHandle(message);    // 后置：AI 回复
    } catch (Exception e) {
        log.error("do handle message operate failed.", e);
    }
}
```

#### 2. 事件监听

**Version 5：**

```java
@EventListener
public void onMessageSent(MessageSentEvent event) {
    ChatMessage message = event.getMessage();
    redisMessageService.publishMessage(message);
}
```

**企业版：**

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
public void onChatSend(ZpWsMessageSendEvent event) throws Exception {
    ZpJobChatWebSocketMessage message = event.getMessage();
    zpJobChatWebSocketService.publish(message);
}
```

#### 3. 分布式消息

**Version 5：**

```java
public void publishMessage(ChatMessage message) {
    String channel = "websocket:user:" + message.getRecipientId();
    redisTemplate.convertAndSend(channel, json);
}
```

**企业版：**

```java
public void publish(T message) throws Exception {
    if (BroadcastType.Redisson.equals(this.manager.getBroadcastType())) {
        this.manager.getRedissonUtils().getTopic(config.getTopicName()).publish(message);
    } else if (BroadcastType.Rabbit.equals(this.manager.getBroadcastType())) {
        this.manager.getRabbitUtils().send(this.config.getExchange(), this.config.getRoutingKey(), message);
    }
}
```

---

## 💼 面试要点

### 1. 架构设计

**Q: 你们的 IM 系统是怎么设计的？**

A: 我们采用分层架构，从下到上分为：
1. **接入层**：管理 WebSocket 连接，用户认证
2. **会话管理层**：管理所有连接，支持多端登录
3. **业务服务层**：消息解析、权限校验、业务处理
4. **策略层**：使用策略模式处理不同业务类型的消息
5. **事件层**：使用 Spring Event 解耦业务逻辑
6. **分布式层**：使用 Redis Pub/Sub 或 RabbitMQ 实现跨服务器通信

每一层职责单一，易于维护和扩展。

---

### 2. 策略模式

**Q: 为什么要用策略模式？**

A: 因为我们有多种业务类型的消息，每种类型的处理逻辑不同：
- 招聘聊天：需要管理聊天室、标签、AI 回复
- 系统通知：只需要推送消息
- 其他业务：有各自的处理逻辑

使用策略模式的好处：
1. **避免 if-else 地狱**：不需要写一个超长的方法
2. **符合开闭原则**：添加新类型只需要新建一个策略类
3. **职责单一**：每个策略类只负责一种业务类型
4. **易于测试**：可以单独测试每个策略类

我们使用工厂模式管理所有策略，Spring 启动时自动扫描注册。

---

### 3. 模板方法模式

**Q: 策略类为什么要用抽象类而不是接口？**

A: 因为我们需要定义处理流程，使用模板方法模式：

```java
public void doHandleMessageOperate(Message message) {
    beforeHandle(message);   // 前置处理
    operateHandle(message);  // 核心处理
    afterHandle(message);    // 后置处理
}
```

这样的好处：
1. **统一流程**：所有策略都按照 before → operate → after 的顺序执行
2. **灵活扩展**：子类可以选择实现哪些方法
3. **共享代码**：抽象类可以有工具方法，子类可以复用

比如招聘聊天：
- before：创建/切换聊天室
- operate：保存消息、更新标签
- after：AI 自动回复

---

### 4. 事件驱动

**Q: 为什么要用事件驱动？**

A: 事件驱动可以解耦业务逻辑和消息推送：

**传统方式（紧耦合）：**
```java
public void handle(Message message) {
    repository.save(message);
    pushToRecipient(message);
    sendNotification(message);
    logMessage(message);
    updateStatistics(message);
}
```
所有逻辑都在一个方法里，添加新功能需要修改这个方法。

**事件驱动（松耦合）：**
```java
public void handle(Message message) {
    repository.save(message);
    eventPublisher.publishEvent(new MessageSentEvent(message));
}

@EventListener
public void onMessageSent(MessageSentEvent event) {
    pushToRecipient(event.getMessage());
}
```
核心逻辑只负责保存消息，其他功能通过监听器实现。

好处：
1. **解耦**：核心逻辑和扩展功能分离
2. **扩展**：添加新功能只需要添加监听器
3. **维护**：每个监听器职责单一

我们还使用了 @TransactionalEventListener，确保事件在事务提交后触发，避免消息推送成功但数据未保存的问题。

---

### 5. 分布式架构

**Q: 如何支持分布式部署？**

A: 我们使用 Redis Pub/Sub 或 RabbitMQ 实现跨服务器通信：

**问题：**
- 用户A连接到服务器1
- 用户B连接到服务器2
- 用户A给用户B发消息，服务器1找不到用户B的连接

**解决方案：**
1. 服务器1收到消息，保存到数据库
2. 发布到 Redis 频道 "user:B"
3. Redis 广播给所有订阅了 "user:B" 的服务器
4. 服务器2收到消息，推送给用户B

我们封装了 TopicManager 和 AbstractTopicService，支持 Redis 和 RabbitMQ 两种方式，可以根据项目需求选择。

---

### 6. 事务管理

**Q: 如何保证消息不丢失？**

A: 我们使用多级事务和事务感知事件：

1. **主事务**：保存消息、更新标签
2. **独立事务**：保存消息状态（REQUIRES_NEW）
3. **事务感知事件**：事务提交后触发消息推送

即使主事务回滚，消息状态也已保存，方便排查问题。

如果消息推送失败，我们会：
1. 保存失败状态到数据库
2. 用户上线后，查询未读消息
3. 重新推送

生产环境还会使用消息队列（RabbitMQ）持久化消息，保证消息可靠性。

---

### 7. 性能优化

**Q: 如何优化性能？**

A: 我们从多个方面优化：

1. **线程安全**：使用 ConcurrentHashMap，减少锁竞争
2. **连接池**：数据库连接池、Redis 连接池
3. **缓存**：用户信息、聊天室信息使用 Redis 缓存
4. **异步处理**：事件监听器可以配置异步执行
5. **批量操作**：批量推送消息、批量更新数据库
6. **分布式部署**：水平扩展，负载均衡

我们还做了压力测试，单机可以支持 1-2 万并发连接，3 台服务器可以支持 3-6 万并发连接。

---

### 8. 高可用

**Q: 如何保证高可用？**

A: 我们从多个方面保证高可用：

1. **服务器集群**：部署多台服务器，负载均衡
2. **Redis 集群**：Redis 主从复制 + 哨兵模式
3. **数据库主从**：MySQL 主从复制，读写分离
4. **消息队列**：RabbitMQ 集群，消息持久化
5. **监控告警**：实时监控服务器状态，异常告警
6. **降级策略**：AI 服务失败不影响主流程

某台服务器宕机，其他服务器继续服务，用户只需要重新连接。

---

## 📚 总结

### 核心技术点

1. **分层架构**：接入层、会话管理层、业务层、策略层、事件层、分布式层
2. **设计模式**：策略模式、工厂模式、模板方法模式、发布订阅模式
3. **事件驱动**：Spring Event、事务感知事件
4. **分布式**：Redis Pub/Sub、RabbitMQ
5. **事务管理**：多级事务、事务传播
6. **性能优化**：线程安全、连接池、缓存、异步处理
7. **高可用**：集群部署、主从复制、监控告警

### 与 Version 5 的差异

1. **业务复杂度**：企业版有聊天室、标签、AI 等复杂业务
2. **架构层次**：企业版分层更清晰，职责更单一
3. **设计模式**：企业版使用模板方法模式
4. **事务管理**：企业版有完整的事务管理
5. **数据库设计**：企业版多表设计，更规范

### 学习建议

1. **先理解 Version 5**：掌握基础架构
2. **对比企业版**：理解为什么企业版更复杂
3. **深入细节**：理解每个组件的实现
4. **实践应用**：尝试添加新功能
5. **准备面试**：整理技术亮点

你已经完全掌握了企业级 IM 系统的核心技术！💪

