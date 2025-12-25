# MallChat 分布式登录模块详解

> 本文档详细讲解 MallChat 项目在分布式环境下，如何实现微信扫码登录的完整流程和稳定性保障机制。

---

## 目录

1. [背景和核心问题](#1-背景和核心问题)
2. [系统架构](#2-系统架构)
3. [完整登录流程](#3-完整登录流程)
4. [核心组件详解](#4-核心组件详解)
5. [分布式问题及解决方案](#5-分布式问题及解决方案)
6. [代码实现详解](#6-代码实现详解)
7. [稳定性保障机制](#7-稳定性保障机制)
8. [常见问题 FAQ](#8-常见问题-faq)

---

## 1. 背景和核心问题

### 1.1 什么是分布式部署？

MallChat 虽然是单体应用，但支持多实例部署：

```
┌─────────────────────────────────────────────┐
│              Nginx (负载均衡)                │
└─────────────────────────────────────────────┘
         │              │              │
         ↓              ↓              ↓
    ┌────────┐    ┌────────┐    ┌────────┐
    │服务器1  │    │服务器2  │    │服务器3  │
    │mallchat│    │mallchat│    │mallchat│
    │  :8080 │    │  :8080 │    │  :8080 │
    └────────┘    └────────┘    └────────┘
```

**说明**：
- 3台服务器运行相同的 mallchat.jar
- Nginx 将用户请求分发到不同服务器
- 每个服务器独立处理请求

### 1.2 核心问题

**问题场景**：
```
1. 用户在浏览器打开网页，请求到达 服务器1
2. 服务器1 生成二维码，用户的 WebSocket 连接在 服务器1
3. 用户用微信扫码，微信回调可能到达 服务器2
4. 服务器2 收到登录成功的消息，但用户的连接在 服务器1
5. 如何让 服务器2 通知到 服务器1，让 服务器1 推送消息给用户？
```

**这就是分布式环境下的核心问题！**

---

## 2. 系统架构

### 2.1 整体架构图

```
┌──────────────────────────────────────────────────────────────┐
│                         用户浏览器                            │
│                    (WebSocket 连接)                          │
└──────────────────────────────────────────────────────────────┘
                              │
                              ↓
┌──────────────────────────────────────────────────────────────┐
│                      Nginx 负载均衡                           │
└──────────────────────────────────────────────────────────────┘
         │                    │                    │
         ↓                    ↓                    ↓
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  服务器1      │    │  服务器2      │    │  服务器3      │
│              │    │              │    │              │
│ - WebSocket  │    │ - WebSocket  │    │ - WebSocket  │
│ - 本地缓存    │    │ - 本地缓存    │    │ - 本地缓存    │
└──────────────┘    └──────────────┘    └──────────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ↓
         ┌────────────────────────────────────────┐
         │         RocketMQ (消息队列)             │
         │         - 广播模式                      │
         └────────────────────────────────────────┘
                              ↓
         ┌────────────────────────────────────────┐
         │         Redis (中心化存储)              │
         │         - Token 存储                    │
         │         - openId -> loginCode 映射      │
         └────────────────────────────────────────┘
                              ↓
         ┌────────────────────────────────────────┐
         │         MySQL (持久化存储)              │
         │         - 用户信息                      │
         └────────────────────────────────────────┘
```

### 2.2 核心组件

| 组件 | 作用 | 为什么需要 |
|------|------|-----------|
| **Nginx** | 负载均衡 | 将请求分发到不同服务器 |
| **本地缓存 (Caffeine)** | 存储 WebSocket 连接 | 快速查找用户连接 |
| **RocketMQ** | 消息队列 | 跨服务器通信 |
| **Redis** | 中心化缓存 | 共享数据（Token、映射关系） |
| **MySQL** | 数据库 | 持久化用户信息 |

---

## 3. 完整登录流程

### 3.1 流程图

```
用户          服务器1         服务器2         RocketMQ        Redis        微信
 │               │              │               │             │            │
 │──1.打开网页──→│              │               │             │            │
 │               │              │               │             │            │
 │               │──2.生成loginCode──────────────→│             │            │
 │               │              │               │             │            │
 │               │──3.存储到本地缓存─────────────→│             │            │
 │               │   (loginCode->channel)        │             │            │
 │               │              │               │             │            │
 │               │──4.请求微信生成二维码─────────────────────────────────→│
 │               │              │               │             │            │
 │←─5.返回二维码─│              │               │             │            │
 │               │              │               │             │            │
 │──────────────6.扫描二维码──────────────────────────────────────────→│
 │               │              │               │             │            │
 │               │              │←─7.微信回调(可能到任意服务器)──────────────│
 │               │              │               │             │            │
 │               │              │──8.查询Redis──────────────→│            │
 │               │              │   (openId->loginCode)       │            │
 │               │              │               │             │            │
 │               │              │──9.发送MQ广播─→│             │            │
 │               │              │   (登录消息)   │             │            │
 │               │              │               │             │            │
 │               │←─10.收到MQ消息────────────────│             │            │
 │               │              │               │             │            │
 │               │──11.查找本地缓存──────────────→│             │            │
 │               │   (找到channel!)              │             │            │
 │               │              │               │             │            │
 │               │──12.生成Token─────────────────────────────→│            │
 │               │              │               │             │            │
 │←─13.推送登录成功─│              │               │             │            │
 │   (包含Token)  │              │               │             │            │
```

### 3.2 详细步骤说明

#### 步骤 1-2：用户请求二维码

**用户操作**：打开网页，点击"微信登录"

**服务器1 处理**：
```java
// 用户请求登录，生成唯一的 loginCode
Integer loginCode = generateLoginCode(channel);
// loginCode 示例：12345
```

**为什么需要 loginCode？**
- 用于关联"二维码"和"用户的 WebSocket 连接"
- 微信扫码后，通过 loginCode 找到对应的用户连接


#### 步骤 3：存储到本地缓存

**服务器1 执行**：
```java
// 将 loginCode 和 channel 的映射关系存储到本地缓存
WAIT_LOGIN_MAP.put(loginCode, channel);

// 数据结构示例：
// loginCode: 12345 -> channel: Channel对象(用户的WebSocket连接)
```

**为什么用本地缓存？**
- **快速查找**：不需要访问 Redis，性能更好
- **只需本地**：channel 对象无法序列化，不能存 Redis
- **自动过期**：1小时后自动清理，避免内存泄漏

**本地缓存配置**：
```java
public static final Cache<Integer, Channel> WAIT_LOGIN_MAP = 
    Caffeine.newBuilder()
        .expireAfterWrite(Duration.ofHours(1))  // 1小时过期
        .maximumSize(10000L)                     // 最多存1万个
        .build();
```

#### 步骤 4-5：生成二维码

**服务器1 执行**：
```java
// 调用微信 API，生成带 loginCode 的二维码
WxMpQrCodeTicket ticket = wxMpService.getQrcodeService()
    .qrCodeCreateTmpTicket(loginCode, 3600);  // 3600秒有效期

// 返回二维码 URL 给前端
String qrCodeUrl = ticket.getUrl();
```

**二维码内容**：
```
https://mp.weixin.qq.com/cgi-bin/showqrcode?ticket=xxx&scene=12345
                                                        ↑
                                                   loginCode
```

**用户看到**：一个二维码图片

#### 步骤 6：用户扫码

**用户操作**：用微信扫描二维码

**微信服务器处理**：
1. 识别二维码中的 `scene=12345`（loginCode）
2. 向 MallChat 服务器发送扫码事件

**关键**：微信的请求可能到达任意一台服务器（由 Nginx 负载均衡决定）


#### 步骤 7：微信回调（关键步骤）

**假设微信回调到达服务器2**（不是生成二维码的服务器1）

**服务器2 收到**：
```java
// 微信发送的扫码事件
WxMpXmlMessage message = {
    fromUser: "oL1Fu5xxxxx",  // 用户的 openId
    eventKey: "qrscene_12345"  // 包含 loginCode
}
```

**服务器2 处理**：
```java
public WxMpXmlOutMessage scan(WxMpXmlMessage wxMpXmlMessage) {
    // 1. 提取 openId 和 loginCode
    String openid = wxMpXmlMessage.getFromUser();  // "oL1Fu5xxxxx"
    Integer loginCode = Integer.parseInt(getEventKey(wxMpXmlMessage));  // 12345
    
    // 2. 查询数据库，看用户是否已注册
    User user = userDao.getByOpenId(openid);
    
    // 3. 如果已注册且有头像，直接登录
    if (Objects.nonNull(user) && StringUtils.isNotEmpty(user.getAvatar())) {
        // 发送登录成功消息到 MQ（重点！）
        mqProducer.sendMsg(
            MQConstant.LOGIN_MSG_TOPIC, 
            new LoginMessageDTO(user.getId(), loginCode)
        );
        return null;
    }
    
    // 4. 如果未注册，先注册
    if (Objects.isNull(user)) {
        user = User.builder().openId(openid).build();
        userService.register(user);
    }
    
    // 5. 存储 openId -> loginCode 映射到 Redis
    RedisUtils.set(
        RedisKey.getKey(RedisKey.OPEN_ID_STRING, openid), 
        loginCode, 
        60, TimeUnit.MINUTES
    );
    
    // 6. 发送扫码成功消息（通知前端等待授权）
    mqProducer.sendMsg(
        MQConstant.SCAN_MSG_TOPIC, 
        new ScanSuccessMessageDTO(loginCode)
    );
    
    // 7. 返回授权链接给用户
    return buildAuthMessage();
}
```

**为什么要存 Redis？**
- 服务器2 不知道 loginCode 对应的 channel 在哪（在服务器1）
- 后续授权时，需要通过 openId 找到 loginCode


#### 步骤 8：查询 Redis

**为什么需要这一步？**

假设用户需要授权（首次登录），微信会再次回调：

```java
@GetMapping("/callBack")
public RedirectView callBack(@RequestParam String code) {
    // 1. 通过 code 获取用户信息
    WxOAuth2AccessToken accessToken = wxService.getOAuth2Service()
        .getAccessToken(code);
    WxOAuth2UserInfo userInfo = wxService.getOAuth2Service()
        .getUserInfo(accessToken, "zh_CN");
    
    // 2. 处理授权
    wxMsgService.authorize(userInfo);
}

public void authorize(WxOAuth2UserInfo userInfo) {
    // 1. 更新用户信息（头像、昵称等）
    User user = userDao.getByOpenId(userInfo.getOpenid());
    fillUserInfo(user.getId(), userInfo);
    
    // 2. 从 Redis 查询 loginCode（关键！）
    Integer code = RedisUtils.get(
        RedisKey.getKey(RedisKey.OPEN_ID_STRING, userInfo.getOpenid()), 
        Integer.class
    );
    // code = 12345
    
    // 3. 发送登录成功消息到 MQ
    mqProducer.sendMsg(
        MQConstant.LOGIN_MSG_TOPIC, 
        new LoginMessageDTO(user.getId(), code)
    );
}
```

**数据流转**：
```
openId: "oL1Fu5xxxxx" 
    ↓ (查询 Redis)
loginCode: 12345
    ↓ (发送 MQ)
通知所有服务器
```

#### 步骤 9：发送 MQ 广播（核心机制）

**服务器2 发送消息**：
```java
mqProducer.sendMsg(
    MQConstant.LOGIN_MSG_TOPIC,  // Topic: "user_login_send_msg"
    new LoginMessageDTO(userId, loginCode)
);

// 消息内容：
// {
//   "uid": 10001,
//   "code": 12345
// }
```

**RocketMQ 配置（广播模式）**：
```java
@RocketMQMessageListener(
    consumerGroup = MQConstant.LOGIN_MSG_GROUP,
    topic = MQConstant.LOGIN_MSG_TOPIC,
    messageModel = MessageModel.BROADCASTING  // ⭐ 广播模式
)
```

**广播模式的作用**：
- **所有服务器实例都会收到这条消息**
- 服务器1、服务器2、服务器3 都会执行消费逻辑
- 不需要知道用户连接在哪台服务器


#### 步骤 10-11：所有服务器收到消息并查找本地缓存

**所有服务器都执行**：
```java
@Component
public class MsgLoginConsumer implements RocketMQListener<LoginMessageDTO> {
    
    @Autowired
    private WebSocketService webSocketService;
    
    @Override
    public void onMessage(LoginMessageDTO loginMessageDTO) {
        // 尝试登录
        webSocketService.scanLoginSuccess(
            loginMessageDTO.getCode(),  // 12345
            loginMessageDTO.getUid()    // 10001
        );
    }
}
```

**每个服务器的处理**：
```java
@Override
public Boolean scanLoginSuccess(Integer loginCode, Long uid) {
    // 1. 查找本地缓存
    Channel channel = WAIT_LOGIN_MAP.getIfPresent(loginCode);
    
    // 2. 如果本地没有这个 channel，返回 false
    if (Objects.isNull(channel)) {
        return Boolean.FALSE;
    }
    
    // 3. 找到了！执行登录逻辑
    User user = userDao.getById(uid);
    WAIT_LOGIN_MAP.invalidate(loginCode);  // 移除缓存
    String token = loginService.login(uid);
    loginSuccess(channel, user, token);
    return Boolean.TRUE;
}
```

**执行结果**：
- **服务器1**：找到了 channel（loginCode: 12345 存在），执行登录 ✅
- **服务器2**：没找到 channel，返回 false ❌
- **服务器3**：没找到 channel，返回 false ❌

**这就是广播模式的精髓**：
- 不需要知道用户在哪
- 所有服务器都尝试
- 只有持有连接的服务器能成功

#### 步骤 12：生成 Token

**服务器1 执行**：
```java
@Override
public String login(Long uid) {
    String key = RedisKey.getKey(RedisKey.USER_TOKEN_STRING, uid);
    // key = "user:token:10001"
    
    // 1. 先查 Redis，避免重复生成
    String token = RedisUtils.getStr(key);
    if (StrUtil.isNotBlank(token)) {
        return token;  // 已有 token，直接返回
    }
    
    // 2. 生成新 token
    token = jwtUtils.createToken(uid);
    // token = "eyJhbGciOiJIUzI1NiJ9.eyJ1aWQiOjEwMDAxfQ.xxx"
    
    // 3. 存入 Redis（5天过期）
    RedisUtils.set(key, token, 5, TimeUnit.DAYS);
    
    return token;
}
```

**为什么用 Redis 存储 Token？**
- **中心化管理**：所有服务器共享同一个 token
- **避免重复**：同一用户不会生成多个 token
- **统一验证**：任意服务器都能验证 token


#### 步骤 13：推送登录成功

**服务器1 执行**：
```java
private void loginSuccess(Channel channel, User user, String token) {
    // 1. 更新在线列表
    online(channel, user.getId());
    
    // 2. 构建登录成功响应
    WSBaseResp<?> resp = WSAdapter.buildLoginSuccessResp(user, token, hasPower);
    
    // 3. 通过 WebSocket 推送给用户
    sendMsg(channel, resp);
    
    // 4. 发布用户上线事件
    applicationEventPublisher.publishEvent(new UserOnlineEvent(this, user));
}

private void sendMsg(Channel channel, WSBaseResp<?> wsBaseResp) {
    // 将消息转为 JSON 并发送
    channel.writeAndFlush(
        new TextWebSocketFrame(JSONUtil.toJsonStr(wsBaseResp))
    );
}
```

**用户收到**：
```json
{
  "type": 2,
  "data": {
    "uid": 10001,
    "name": "张三",
    "avatar": "https://xxx.jpg",
    "token": "eyJhbGciOiJIUzI1NiJ9.eyJ1aWQiOjEwMDAxfQ.xxx"
  }
}
```

**前端处理**：
```javascript
// 收到登录成功消息
websocket.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.type === 2) {  // 登录成功
    // 保存 token
    localStorage.setItem('token', data.data.token);
    // 跳转到聊天页面
    router.push('/chat');
  }
};
```

---

## 4. 核心组件详解

### 4.1 本地缓存 (Caffeine)

**作用**：存储 loginCode 和 WebSocket Channel 的映射关系

**为什么不用 Redis？**
1. **Channel 对象无法序列化**：WebSocket 连接对象不能存到 Redis
2. **性能更好**：本地内存访问比网络访问快
3. **只需本地**：Channel 只在当前服务器有效

**配置详解**：
```java
public static final Cache<Integer, Channel> WAIT_LOGIN_MAP = 
    Caffeine.newBuilder()
        .expireAfterWrite(Duration.ofHours(1))  // 写入后1小时过期
        .maximumSize(10000L)                     // 最多存储1万个
        .build();
```

**参数说明**：
- `expireAfterWrite(1小时)`：防止用户请求二维码后不扫码，占用内存
- `maximumSize(10000)`：限制内存使用，超过后自动淘汰旧数据

**使用示例**：
```java
// 存储
WAIT_LOGIN_MAP.put(12345, channel);

// 查询
Channel channel = WAIT_LOGIN_MAP.getIfPresent(12345);

// 删除
WAIT_LOGIN_MAP.invalidate(12345);
```


### 4.2 RocketMQ 广播模式

**作用**：实现跨服务器通信

**为什么用广播模式？**

**对比两种模式**：

| 模式 | 消息分发 | 适用场景 |
|------|---------|---------|
| **集群模式 (CLUSTERING)** | 一条消息只被一个消费者消费 | 任务分发、负载均衡 |
| **广播模式 (BROADCASTING)** | 一条消息被所有消费者消费 | 通知、缓存刷新 |

**登录场景为什么用广播？**

```
问题：不知道用户连接在哪台服务器

集群模式：
服务器2 发送消息 → MQ → 随机分配给服务器1
                        ↓
                    如果用户在服务器3，就失败了 ❌

广播模式：
服务器2 发送消息 → MQ → 服务器1 收到 ✅
                     ↓→ 服务器2 收到 ✅
                     ↓→ 服务器3 收到 ✅
                        ↓
                    总有一个服务器能找到用户 ✅
```

**配置详解**：
```java
@RocketMQMessageListener(
    consumerGroup = "user_login_send_msg_group",  // 消费者组
    topic = "user_login_send_msg",                // 主题
    messageModel = MessageModel.BROADCASTING      // 广播模式 ⭐
)
public class MsgLoginConsumer implements RocketMQListener<LoginMessageDTO> {
    @Override
    public void onMessage(LoginMessageDTO message) {
        // 所有服务器都会执行这里的代码
    }
}
```

**消息流转**：
```
服务器2 发送消息
    ↓
RocketMQ Broker
    ↓
┌───┴───┬───────┬───────┐
↓       ↓       ↓       ↓
服务器1  服务器2  服务器3  服务器4
(都收到) (都收到) (都收到) (都收到)
```

### 4.3 Redis 中心化存储

**作用**：存储需要跨服务器共享的数据

**存储内容**：

#### 1. Token 存储
```java
// Key: user:token:{uid}
// Value: JWT token
// 过期时间: 5天

RedisUtils.set(
    "user:token:10001", 
    "eyJhbGciOiJIUzI1NiJ9.xxx",
    5, TimeUnit.DAYS
);
```

**为什么存 Redis？**
- 所有服务器共享同一个 token
- 避免同一用户生成多个 token
- 统一管理 token 过期


#### 2. openId -> loginCode 映射
```java
// Key: openId:{openId}
// Value: loginCode
// 过期时间: 60分钟

RedisUtils.set(
    "openId:oL1Fu5xxxxx", 
    12345,
    60, TimeUnit.MINUTES
);
```

**为什么需要这个映射？**

```
场景：用户扫码后需要授权

1. 扫码事件：服务器2 收到，知道 loginCode=12345
2. 授权回调：服务器3 收到，只知道 openId="oL1Fu5xxxxx"
3. 问题：服务器3 如何知道 loginCode？
4. 解决：通过 Redis 查询 openId -> loginCode
```

#### 3. loginCode 自增
```java
// Key: loginCode
// Value: 自增数字
// 过期时间: 1小时

int loginCode = RedisUtils.integerInc(
    "loginCode", 
    60, TimeUnit.MINUTES
);
```

**为什么用 Redis 自增？**
- 保证 loginCode 全局唯一
- 避免多个服务器生成相同的 loginCode

**自增逻辑**：
```java
private Integer generateLoginCode(Channel channel) {
    int inc;
    do {
        // Redis 自增，返回新值
        inc = RedisUtils.integerInc("loginCode", 60, TimeUnit.MINUTES);
        // 检查本地缓存是否已存在（避免冲突）
    } while (WAIT_LOGIN_MAP.asMap().containsKey(inc));
    
    // 存储到本地缓存
    WAIT_LOGIN_MAP.put(inc, channel);
    return inc;
}
```

---

## 5. 分布式问题及解决方案

### 5.1 问题1：WebSocket 连接分散

**问题描述**：
```
用户A 连接到 服务器1
用户B 连接到 服务器2
用户C 连接到 服务器3

如何让用户A给用户B发消息？
```

**解决方案**：MQ 广播 + 本地查找

```java
// 发送消息流程
1. 用户A 发送消息给用户B
2. 服务器1 收到消息，发送到 MQ
3. 所有服务器收到 MQ 消息
4. 服务器2 发现用户B在本地，推送消息
5. 服务器1、3 发现用户B不在本地，忽略
```

**代码实现**：
```java
// 推送消息给指定用户
@Override
public void sendToUid(WSBaseResp<?> wsBaseResp, Long uid) {
    // 1. 查找本地连接
    CopyOnWriteArrayList<Channel> channels = ONLINE_UID_MAP.get(uid);
    
    // 2. 如果不在本地，记录日志
    if (CollectionUtil.isEmpty(channels)) {
        log.info("用户：{}不在线", uid);
        return;
    }
    
    // 3. 推送给本地的所有连接（支持多端登录）
    channels.forEach(channel -> {
        threadPoolTaskExecutor.execute(() -> sendMsg(channel, wsBaseResp));
    });
}
```


### 5.2 问题2：Token 一致性

**问题描述**：
```
用户在服务器1登录，生成 token1
用户在服务器2刷新页面，生成 token2
结果：用户有两个不同的 token，导致混乱
```

**解决方案**：Redis 中心化存储

```java
@Override
public String login(Long uid) {
    String key = "user:token:" + uid;
    
    // 1. 先查 Redis
    String token = RedisUtils.getStr(key);
    if (StrUtil.isNotBlank(token)) {
        return token;  // 已有 token，直接返回 ✅
    }
    
    // 2. 没有 token，生成新的
    token = jwtUtils.createToken(uid);
    
    // 3. 存入 Redis
    RedisUtils.set(key, token, 5, TimeUnit.DAYS);
    
    return token;
}
```

**效果**：
```
服务器1：查询 Redis → 没有 → 生成 token1 → 存入 Redis
服务器2：查询 Redis → 有 token1 → 直接返回 token1 ✅
```

### 5.3 问题3：重复登录

**问题描述**：
```
用户扫码后，网络不好，微信重试了3次回调
结果：收到3次登录消息，推送3次
```

**解决方案1**：登录成功后立即删除缓存

```java
@Override
public Boolean scanLoginSuccess(Integer loginCode, Long uid) {
    Channel channel = WAIT_LOGIN_MAP.getIfPresent(loginCode);
    if (Objects.isNull(channel)) {
        return Boolean.FALSE;
    }
    
    // 立即删除缓存 ⭐
    WAIT_LOGIN_MAP.invalidate(loginCode);
    
    // 执行登录
    loginSuccess(channel, user, token);
    return Boolean.TRUE;
}
```

**效果**：
```
第1次：找到 channel → 删除缓存 → 登录成功 ✅
第2次：找不到 channel → 返回 false ✅
第3次：找不到 channel → 返回 false ✅
```

**解决方案2**：Token 复用

```java
// 同一用户多次登录，返回同一个 token
String token = RedisUtils.getStr(key);
if (StrUtil.isNotBlank(token)) {
    return token;  // 复用 token
}
```

### 5.4 问题4：loginCode 冲突

**问题描述**：
```
服务器1 生成 loginCode=12345
服务器2 也生成 loginCode=12345
结果：两个用户的二维码相同，登录混乱
```

**解决方案**：Redis 自增 + 本地检查

```java
private Integer generateLoginCode(Channel channel) {
    int inc;
    do {
        // Redis 自增，保证全局唯一 ⭐
        inc = RedisUtils.integerInc("loginCode", 60, TimeUnit.MINUTES);
        
        // 本地检查，避免缓存未过期的冲突
    } while (WAIT_LOGIN_MAP.asMap().containsKey(inc));
    
    WAIT_LOGIN_MAP.put(inc, channel);
    return inc;
}
```

**Redis 自增原理**：
```
服务器1：INCR loginCode → 返回 1
服务器2：INCR loginCode → 返回 2
服务器3：INCR loginCode → 返回 3
```


### 5.5 问题5：用户名重复

**问题描述**：
```
多个用户同时授权，微信昵称都是"张三"
数据库有唯一索引，插入失败
```

**解决方案**：重试 + 随机后缀

```java
private void fillUserInfo(Long uid, WxOAuth2UserInfo userInfo) {
    User update = UserAdapter.buildAuthorizeUser(uid, userInfo);
    
    // 重试5次
    for (int i = 0; i < 5; i++) {
        try {
            userDao.updateById(update);
            return;  // 成功则返回
        } catch (DuplicateKeyException e) {
            // 用户名重复，添加随机后缀
            log.info("fill userInfo duplicate uid:{}", uid);
        } catch (Exception e) {
            log.error("fill userInfo fail uid:{}", uid);
        }
        
        // 修改用户名，重试
        update.setName("名字重置" + RandomUtil.randomInt(100000));
    }
}
```

**效果**：
```
第1次：插入"张三" → 失败（重复）
第2次：插入"名字重置12345" → 成功 ✅
```

---

## 6. 代码实现详解

### 6.1 WebSocket 连接管理

**数据结构**：
```java
// 1. 所有 WebSocket 连接
private static final ConcurrentHashMap<Channel, WSChannelExtraDTO> ONLINE_WS_MAP 
    = new ConcurrentHashMap<>();

// 2. 用户ID -> 连接列表（支持多端登录）
private static final ConcurrentHashMap<Long, CopyOnWriteArrayList<Channel>> ONLINE_UID_MAP 
    = new ConcurrentHashMap<>();
```

**为什么用 ConcurrentHashMap？**
- 线程安全
- 高并发性能好

**为什么用 CopyOnWriteArrayList？**
- 支持多端登录（同一用户多个连接）
- 读多写少的场景，性能好

**连接建立**：
```java
@Override
public void connect(Channel channel) {
    // 存储连接
    ONLINE_WS_MAP.put(channel, new WSChannelExtraDTO());
}
```

**用户上线**：
```java
private void online(Channel channel, Long uid) {
    // 1. 设置 channel 的 uid
    getOrInitChannelExt(channel).setUid(uid);
    
    // 2. 添加到用户连接列表
    ONLINE_UID_MAP.putIfAbsent(uid, new CopyOnWriteArrayList<>());
    ONLINE_UID_MAP.get(uid).add(channel);
    
    // 3. 设置 channel 属性
    NettyUtil.setAttr(channel, NettyUtil.UID, uid);
}
```

**用户下线**：
```java
private boolean offline(Channel channel, Optional<Long> uidOptional) {
    // 1. 移除连接
    ONLINE_WS_MAP.remove(channel);
    
    // 2. 如果已登录，移除用户连接列表
    if (uidOptional.isPresent()) {
        CopyOnWriteArrayList<Channel> channels = ONLINE_UID_MAP.get(uidOptional.get());
        if (CollectionUtil.isNotEmpty(channels)) {
            channels.removeIf(ch -> Objects.equals(ch, channel));
        }
        // 返回是否全部下线
        return CollectionUtil.isEmpty(ONLINE_UID_MAP.get(uidOptional.get()));
    }
    return true;
}
```


### 6.2 Token 验证流程

**验证 Token**：
```java
@Override
public boolean verify(String token) {
    // 1. 解析 token，获取 uid
    Long uid = jwtUtils.getUidOrNull(token);
    if (Objects.isNull(uid)) {
        return false;  // token 无效
    }
    
    // 2. 从 Redis 获取真实 token
    String key = RedisKey.getKey(RedisKey.USER_TOKEN_STRING, uid);
    String realToken = RedisUtils.getStr(key);
    
    // 3. 比对 token 是否一致
    return Objects.equals(token, realToken);
}
```

**为什么要比对？**
```
场景：用户修改密码后，旧 token 应该失效

1. 用户登录，生成 token1，存入 Redis
2. 用户修改密码，生成 token2，覆盖 Redis
3. 用户用 token1 访问，解析成功，但 Redis 中是 token2
4. 比对失败，token1 失效 ✅
```

**自动续期**：
```java
@Async  // 异步执行，不阻塞主流程
@Override
public void renewalTokenIfNecessary(String token) {
    Long uid = jwtUtils.getUidOrNull(token);
    if (Objects.isNull(uid)) {
        return;
    }
    
    String key = RedisKey.getKey(RedisKey.USER_TOKEN_STRING, uid);
    
    // 获取剩余过期时间
    long expireDays = RedisUtils.getExpire(key, TimeUnit.DAYS);
    
    if (expireDays == -2) {  // key 不存在
        return;
    }
    
    // 剩余时间 < 2天，续期到5天
    if (expireDays < TOKEN_RENEWAL_DAYS) {
        RedisUtils.expire(key, TOKEN_EXPIRE_DAYS, TimeUnit.DAYS);
    }
}
```

**续期策略**：
```
Token 有效期：5天
续期阈值：剩余 < 2天

时间线：
Day 0: 登录，token 有效期5天
Day 1: 访问，剩余4天，不续期
Day 2: 访问，剩余3天，不续期
Day 3: 访问，剩余2天，不续期
Day 4: 访问，剩余1天，续期到5天 ✅
Day 5: 访问，剩余4天，不续期
...
```

### 6.3 消息推送实现

**推送给指定用户**：
```java
@Override
public void sendToUid(WSBaseResp<?> wsBaseResp, Long uid) {
    // 1. 获取用户的所有连接
    CopyOnWriteArrayList<Channel> channels = ONLINE_UID_MAP.get(uid);
    
    // 2. 用户不在线
    if (CollectionUtil.isEmpty(channels)) {
        log.info("用户：{}不在线", uid);
        return;
    }
    
    // 3. 推送给所有连接（支持多端）
    channels.forEach(channel -> {
        // 异步推送，避免阻塞
        threadPoolTaskExecutor.execute(() -> sendMsg(channel, wsBaseResp));
    });
}
```

**推送给所有在线用户**：
```java
@Override
public void sendToAllOnline(WSBaseResp<?> wsBaseResp, Long skipUid) {
    ONLINE_WS_MAP.forEach((channel, ext) -> {
        // 跳过指定用户
        if (Objects.nonNull(skipUid) && Objects.equals(ext.getUid(), skipUid)) {
            return;
        }
        // 异步推送
        threadPoolTaskExecutor.execute(() -> sendMsg(channel, wsBaseResp));
    });
}
```

**发送消息**：
```java
private void sendMsg(Channel channel, WSBaseResp<?> wsBaseResp) {
    // 将对象转为 JSON，通过 WebSocket 发送
    channel.writeAndFlush(
        new TextWebSocketFrame(JSONUtil.toJsonStr(wsBaseResp))
    );
}
```


---

## 7. 稳定性保障机制

### 7.1 缓存过期策略

**多层过期时间**：
```java
// 1. 本地缓存：1小时
WAIT_LOGIN_MAP = Caffeine.newBuilder()
    .expireAfterWrite(Duration.ofHours(1))
    .build();

// 2. Redis openId 映射：60分钟
RedisUtils.set(key, loginCode, 60, TimeUnit.MINUTES);

// 3. Redis loginCode：60分钟
RedisUtils.integerInc("loginCode", 60, TimeUnit.MINUTES);

// 4. Token：5天
RedisUtils.set(key, token, 5, TimeUnit.DAYS);
```

**为什么本地缓存时间要短？**
```
问题：本地缓存时间 > Redis 时间

场景：
1. Redis loginCode 过期（60分钟）
2. 新用户生成相同的 loginCode
3. 旧用户的本地缓存还在（90分钟）
4. 冲突！

解决：本地缓存时间 ≤ Redis 时间
```

### 7.2 异步处理

**为什么要异步？**
```
同步推送：
用户登录 → 推送消息 → 等待发送完成 → 返回
              ↓ (如果网络慢，阻塞主线程)
           耗时 100ms

异步推送：
用户登录 → 提交推送任务 → 立即返回 ✅
              ↓ (后台线程处理)
           耗时 1ms
```

**线程池配置**：
```java
@Bean(ThreadPoolConfig.WS_EXECUTOR)
public ThreadPoolTaskExecutor websocketExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(16);           // 核心线程数
    executor.setMaxPoolSize(16);            // 最大线程数
    executor.setQueueCapacity(1000);        // 队列容量
    executor.setThreadNamePrefix("websocket-executor-");
    
    // 队列满了，直接丢弃（不重要的消息）
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
    
    executor.initialize();
    return executor;
}
```

**使用示例**：
```java
// 异步推送
threadPoolTaskExecutor.execute(() -> sendMsg(channel, wsBaseResp));

// 异步续期
@Async
public void renewalTokenIfNecessary(String token) {
    // ...
}
```

### 7.3 幂等性保证

**什么是幂等性？**
```
同一个操作执行多次，结果相同

例如：
- 登录3次 = 登录1次（返回同一个 token）
- 扣款3次 ≠ 扣款1次（需要防重）
```

**登录的幂等性**：

**方法1：Token 复用**
```java
String token = RedisUtils.getStr(key);
if (StrUtil.isNotBlank(token)) {
    return token;  // 已有 token，直接返回
}
```

**方法2：缓存失效**
```java
// 登录成功后立即删除
WAIT_LOGIN_MAP.invalidate(loginCode);
```

**方法3：Redis 覆盖写入**
```java
// 同一个 openId，覆盖旧的 loginCode
RedisUtils.set(openIdKey, loginCode, 60, TimeUnit.MINUTES);
```


### 7.4 异常处理

**数据库异常重试**：
```java
private void fillUserInfo(Long uid, WxOAuth2UserInfo userInfo) {
    User update = UserAdapter.buildAuthorizeUser(uid, userInfo);
    
    for (int i = 0; i < 5; i++) {
        try {
            userDao.updateById(update);
            return;
        } catch (DuplicateKeyException e) {
            // 用户名重复
            log.info("fill userInfo duplicate uid:{},info:{}", uid, userInfo);
        } catch (Exception e) {
            // 其他异常
            log.error("fill userInfo fail uid:{},info:{}", uid, userInfo);
        }
        // 修改用户名，重试
        update.setName("名字重置" + RandomUtil.randomInt(100000));
    }
}
```

**WebSocket 异常处理**：
```java
@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    log.warn("异常发生，异常消息 ={}", cause);
    ctx.channel().close();  // 关闭连接
}
```

**心跳超时处理**：
```java
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
    if (evt instanceof IdleStateEvent) {
        IdleStateEvent idleStateEvent = (IdleStateEvent) evt;
        // 30秒没收到消息，关闭连接
        if (idleStateEvent.state() == IdleState.READER_IDLE) {
            userOffLine(ctx);
        }
    }
}
```

### 7.5 监控和日志

**关键日志**：
```java
// 1. 用户不在线
log.info("用户：{}不在线", uid);

// 2. 用户名重复
log.info("fill userInfo duplicate uid:{}", uid);

// 3. 更新失败
log.error("fill userInfo fail uid:{}", uid);

// 4. 连接断开
log.warn("触发 channelInactive 掉线![{}]", ctx.channel().id());
```

**建议添加的监控**：
```java
// 1. 登录成功率
// 2. 登录耗时
// 3. Token 验证失败次数
// 4. WebSocket 连接数
// 5. MQ 消息堆积
```

---

## 8. 常见问题 FAQ

### Q1: 为什么不用 Redis 存储 Channel？

**答**：
1. Channel 是 Netty 的连接对象，无法序列化
2. Channel 只在当前 JVM 有效，存 Redis 没意义
3. 本地内存访问更快

### Q2: 广播模式会不会浪费资源？

**答**：
是的，会有一定浪费：
- 所有服务器都收到消息
- 只有一个服务器能处理
- 其他服务器白白消费了消息

**优化方案**：
```java
// 方案1：Redis 存储 loginCode -> serverId
RedisUtils.set("loginCode:12345", "server-1");

// 方案2：精确路由（需要改造 MQ）
mqProducer.sendToServer("server-1", message);
```

**为什么不优化？**
- 登录是低频操作，浪费可接受
- 广播模式实现简单
- 避免 Redis 单点故障


### Q3: 如果 Redis 挂了怎么办？

**答**：
会导致以下问题：
1. 无法生成 loginCode（Redis 自增失败）
2. 无法存储 Token（登录失败）
3. 无法验证 Token（所有请求失败）

**解决方案**：
```java
// 1. Redis 主从 + 哨兵（高可用）
// 2. Redis 集群（高可用 + 高性能）
// 3. 降级方案（本地生成 loginCode）

private Integer generateLoginCode(Channel channel) {
    try {
        return RedisUtils.integerInc("loginCode", 60, TimeUnit.MINUTES);
    } catch (Exception e) {
        // Redis 挂了，使用本地生成
        log.error("Redis 异常，使用本地生成 loginCode", e);
        return generateLocalLoginCode();
    }
}
```

### Q4: 如果 RocketMQ 挂了怎么办？

**答**：
会导致：
1. 扫码后无法通知到持有连接的服务器
2. 用户一直等待，登录失败

**解决方案**：
```java
// 1. RocketMQ 集群（高可用）
// 2. 降级方案（轮询 Redis）

// 用户扫码后，存储到 Redis
RedisUtils.set("login:pending:" + loginCode, uid, 60, TimeUnit.SECONDS);

// 前端轮询接口
@GetMapping("/checkLogin")
public Result checkLogin(Integer loginCode) {
    Long uid = RedisUtils.get("login:pending:" + loginCode, Long.class);
    if (uid != null) {
        // 登录成功
        return Result.success(uid);
    }
    return Result.fail("等待登录");
}
```

### Q5: 如何支持多端登录？

**答**：
已经支持了！

```java
// 用户ID -> 连接列表（支持多个连接）
private static final ConcurrentHashMap<Long, CopyOnWriteArrayList<Channel>> ONLINE_UID_MAP;

// 用户上线时，添加到列表
ONLINE_UID_MAP.get(uid).add(channel);

// 推送消息时，推送给所有连接
channels.forEach(channel -> sendMsg(channel, wsBaseResp));
```

**效果**：
```
用户A：
- 手机端：连接到服务器1
- 电脑端：连接到服务器2
- 平板端：连接到服务器1

推送消息时，3个设备都能收到 ✅
```

### Q6: loginCode 会不会用完？

**答**：
不会，因为：
1. loginCode 是 int 类型，最大值 2^31 - 1 = 2,147,483,647
2. 每小时重置（Redis 过期）
3. 即使每秒生成1000个，也要24天才用完

**计算**：
```
每秒生成：1000个
每小时：1000 * 3600 = 360万
最大值：21亿
可用时间：21亿 / 360万 / 24 = 24天
```

### Q7: 为什么 Token 有效期是5天？

**答**：
这是一个权衡：
- **太短**（1天）：用户频繁登录，体验差
- **太长**（30天）：安全风险高

**5天的好处**：
1. 用户一周内不用重新登录
2. 配合自动续期，长期在线用户不会掉线
3. 不活跃用户自动过期，减少安全风险

**续期策略**：
```
用户每天访问 → 剩余时间 < 2天 → 自动续期到5天
结果：活跃用户永远不会过期 ✅
```


### Q8: 如何防止恶意扫码攻击？

**答**：
需要添加限流机制：

```java
// 1. IP 限流（每个 IP 每分钟最多请求10次）
@FrequencyControl(time = 60, count = 10, target = FrequencyControl.Target.IP)
public void handleLoginReq(Channel channel) {
    // 生成二维码
}

// 2. 用户限流（每个用户每分钟最多扫码5次）
@FrequencyControl(time = 60, count = 5, target = FrequencyControl.Target.UID)
public void scan(WxMpXmlMessage wxMpXmlMessage) {
    // 处理扫码
}
```

**MallChat 已有的限流注解**：
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface FrequencyControl {
    int time();   // 时间窗口（秒）
    int count();  // 最大次数
    Target target();  // 限流目标（IP/UID）
}
```

### Q9: 如何实现扫码登录的超时提示？

**答**：
前端轮询 + 后端超时检测

**前端实现**：
```javascript
// 请求二维码
const loginCode = await getQrCode();

// 轮询检查登录状态（每2秒一次）
const timer = setInterval(async () => {
  const result = await checkLoginStatus(loginCode);
  if (result.success) {
    clearInterval(timer);
    // 登录成功
  }
}, 2000);

// 2分钟后超时
setTimeout(() => {
  clearInterval(timer);
  showMessage('二维码已过期，请刷新');
}, 120000);
```

**后端实现**：
```java
// 本地缓存1小时过期
WAIT_LOGIN_MAP = Caffeine.newBuilder()
    .expireAfterWrite(Duration.ofHours(1))
    .build();

// 查询时检查是否过期
Channel channel = WAIT_LOGIN_MAP.getIfPresent(loginCode);
if (channel == null) {
    return Result.fail("二维码已过期");
}
```

### Q10: 如何优化登录性能？

**答**：
已有的优化：
1. **本地缓存**：避免频繁访问 Redis
2. **异步推送**：不阻塞主线程
3. **Token 复用**：避免重复生成

**可以继续优化**：
```java
// 1. 批量推送（减少网络开销）
List<Channel> channels = new ArrayList<>();
channels.addAll(ONLINE_UID_MAP.get(uid1));
channels.addAll(ONLINE_UID_MAP.get(uid2));
batchSend(channels, message);

// 2. 连接预热（提前建立连接）
@PostConstruct
public void init() {
    // 预热 Redis 连接池
    redisTemplate.opsForValue().get("warmup");
}

// 3. 监控慢查询
@Around("@annotation(LoginService)")
public Object monitor(ProceedingJoinPoint pjp) {
    long start = System.currentTimeMillis();
    Object result = pjp.proceed();
    long cost = System.currentTimeMillis() - start;
    if (cost > 1000) {
        log.warn("登录耗时过长：{}ms", cost);
    }
    return result;
}
```

---

## 9. 总结

### 9.1 核心设计思想

1. **本地优先，Redis 兜底**
   - 本地缓存存储 Channel（快速查找）
   - Redis 存储共享数据（跨服务器）

2. **广播模式解决路由问题**
   - 不需要知道用户在哪个服务器
   - 所有服务器都尝试，只有一个成功

3. **中心化 Token 管理**
   - Redis 作为唯一数据源
   - 避免多服务器生成不同 Token

4. **异步 + 重试保证可靠性**
   - 异步推送不阻塞
   - 重试机制应对异常

### 9.2 关键技术点

| 技术 | 作用 | 为什么选择 |
|------|------|-----------|
| **Caffeine** | 本地缓存 | 快速、自动过期 |
| **RocketMQ** | 消息队列 | 支持广播模式 |
| **Redis** | 中心化存储 | 高性能、支持自增 |
| **JWT** | Token 生成 | 无状态、易验证 |
| **Netty** | WebSocket | 高性能、异步 |

### 9.3 数据流转图

```
┌─────────────────────────────────────────────────────────┐
│                      完整数据流                          │
└─────────────────────────────────────────────────────────┘

1. 生成 loginCode
   服务器1 → Redis (INCR) → 返回 12345

2. 存储映射关系
   服务器1 → 本地缓存 (12345 -> channel)

3. 用户扫码
   微信 → 服务器2 (随机)

4. 存储 openId 映射
   服务器2 → Redis (openId -> 12345)

5. 发送 MQ 广播
   服务器2 → RocketMQ → 所有服务器

6. 查找本地缓存
   服务器1 → 本地缓存 → 找到 channel ✅
   服务器2 → 本地缓存 → 未找到 ❌
   服务器3 → 本地缓存 → 未找到 ❌

7. 生成 Token
   服务器1 → Redis (存储 Token)

8. 推送消息
   服务器1 → WebSocket → 用户
```

### 9.4 适用场景

**适合**：
- 中小型项目（用户 < 100万）
- 登录频率不高（每天 < 10万次）
- 团队规模小（< 10人）

**不适合**：
- 超大规模（用户 > 1000万）
- 高频登录（每秒 > 1000次）
- 需要精确路由的场景

### 9.5 改进方向

1. **精确路由**：用 Redis 存储 `loginCode -> serverId`
2. **限流保护**：防止恶意扫码攻击
3. **监控告警**：登录失败率、耗时监控
4. **降级方案**：Redis/MQ 挂了的兜底策略
5. **安全加固**：Token 加密、防重放攻击

---

## 10. 参考资料

- [MallChat 项目地址](https://github.com/zongzibinbin/MallChat)
- [RocketMQ 官方文档](https://rocketmq.apache.org/)
- [Caffeine 缓存库](https://github.com/ben-manes/caffeine)
- [Netty 官方文档](https://netty.io/)

---

**文档版本**：v1.0  
**最后更新**：2024-12-25  
**作者**：Kiro AI Assistant

不同版本是怎么升级的，能兼容吗？有回滚机制吗？