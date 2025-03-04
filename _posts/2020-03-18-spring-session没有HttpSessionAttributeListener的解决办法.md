---
layout: post
title: "spring-session没有HttpSessionAttributeListener的解决办法"
date: 2020-03-18 08:54:51
categories: 日常工作
---

> spring-session中对于servlet3.0标准中`HttpSessionAttributeListener`监听器没有支持，所以如果想要监听spring-session中session属性的变化就需要自己编码处理，经上网查阅资料与源码，问题得以解决。

<!-- more -->

### 起因

---

- 最近的项目需要统计登录session，但是现有的系统中，用户访问后创建的session不一定是登录的session，只有在session的属性中设置了指定的值才算作登录。servlet3.0中支持`HttpSessionAttributeListener`用来监听session属性的修改，但是蛋疼的是spring-session中对于servlet标准监听器只支持`HttpSessionListener`，所以得自己想办法解决。

### 问题剖析

---

- spring-session的GitHub repository上有人提了[issue](https://github.com/spring-projects/spring-session/issues/5)，spring-session目前还不支持`HttpSessionAttributeListener`，通过网上对于spring-session源码解析的[资料](https://www.cnblogs.com/lxyit/p/9719542.html)，得知spring-session对于`HttpSessionListener`的处理是通过`redis`的[Redis Keyspace Notifications](https://redis.io/topics/notifications)功能实现的。为此，我们可以参考`HttpSessionListener`的实现逻辑，处理下session属性的监听。
- spring-session在创建session后，会持久化session到redis中，对于一个session，会创建3个redis的值，分别为：
  - `spring:session:sessions:+sessionId`：此值为hash，其中存储了session中的属性，还包括session的持续时长、上次访问时间以及创建时间
  - `spring:session:sessions:expires:+sessionId`：此key中没有值，主要是利用这个key的过期时间，确定session是否过期
  - `spring:session:expirations:+时间戳`：此值为set，其中存储了对应时间过期的sessionid，通过源码可以得知，此处所有的时间戳都是整分钟的时间戳，spring-session在处理时会先将session的过期时间取下一个最近的整分钟时间戳，并将sessionId存储到对应的set中。

- spring-session中在操作`redis`时，主要是通过`RedisOperationsSessionRepository`这个类实现的，我们可以在业务代码中注入此类的对象，即可操作session。`RedisOperationsSessionRepository`中涉及到保存session到`redis`的方法是 save() 方法，代码如下：

  ```
  public void save(RedisSession session) {
      session.saveDelta(); // 保存session到redis
      if (session.isNew()) { // session是新建的，需要通过pub/sub发布消息，触发httpSessionListener，不在本篇文章分析的范围内
          String sessionCreatedKey = getSessionCreatedChannel(session.getId());
          this.sessionRedisOperations.convertAndSend(sessionCreatedKey, session.delta);
          session.setNew(false);
      }
  }
  ```

  其中最主要的就是 `session.saveDelta()` 方法，`RedisSession`代码如下：

  ```
  /**
  	 * A custom implementation of {@link Session} that uses a {@link MapSession} as the
  	 * basis for its mapping. It keeps track of any attributes that have changed. When
  	 * {@link org.springframework.session.data.redis.RedisOperationsSessionRepository.RedisSession#saveDelta()}
  	 * is invoked all the attributes that have been changed will be persisted.
  	 *
  	 * @author Rob Winch
  	 * @since 1.0
  	 */
  	final class RedisSession implements Session {
  		private final MapSession cached; // 内存中的session
  		private Instant originalLastAccessTime; // 上次访问时间
  		private Map<String, Object> delta = new HashMap<>(); // 更新的数据
  		private boolean isNew;// 是否发布过创建的消息
  		private String originalPrincipalName; 
  
  		/**
  		 * Creates a new instance ensuring to mark all of the new attributes to be
  		 * persisted in the next save operation.
  		 */
  		RedisSession() {
  			this(new MapSession());
  			// 新建session的时候会默认将创建时间、上次访问时间和session有效期添加到session
  			this.delta.put(CREATION_TIME_ATTR, getCreationTime().toEpochMilli());
  			this.delta.put(MAX_INACTIVE_ATTR, (int) getMaxInactiveInterval().getSeconds());
  			this.delta.put(LAST_ACCESSED_ATTR, getLastAccessedTime().toEpochMilli());
  			this.isNew = true;
  			this.flushImmediateIfNecessary();
  		}
   		...
   		省略无关代码
   		...
  
  		/**
  		 * Saves any attributes that have been changed and updates the expiration of this
  		 * session.
  		 */
  		 // 保存session到redis的方法
  		private void saveDelta() {
  			if (this.delta.isEmpty()) {
  				return;
  			}
  			String sessionId = getId();
  			// 此处为保存session属性，delta为更新的session信息
  			getSessionBoundHashOperations(sessionId).putAll(this.delta);
  			String principalSessionKey = getSessionAttrNameKey(
  			...
  			省略无关代码
  			...
  			this.delta = new HashMap<>(this.delta.size());
  			// 下面是更新过期时间的逻辑
  			Long originalExpiration = this.originalLastAccessTime == null ? null
  					: this.originalLastAccessTime.plus(getMaxInactiveInterval()).toEpochMilli();
  			RedisOperationsSessionRepository.this.expirationPolicy
  					.onExpirationUpdated(originalExpiration, this);
  		}
  	}
  ```

  `getSessionBoundHashOperations`方法代码如下：

  ```
  private String keyPrefix = DEFAULT_SPRING_SESSION_REDIS_PREFIX;
  static final String DEFAULT_SPRING_SESSION_REDIS_PREFIX = "spring:session:";
  
  private BoundHashOperations<Object, Object, Object> getSessionBoundHashOperations(
  			String sessionId) {
  	// 此处拼装得到的就是spring:session:sessions:sessionId
  	String key = getSessionKey(sessionId); 
  	// 操作redis中session对应的set，更新hashkey\hashvalue
  	return this.sessionRedisOperations.boundHashOps(key);
  }
  
  String getSessionKey(String sessionId) {
  	return this.keyPrefix + "sessions:" + sessionId;
  }
  ```

  可以看到，最终session属性变化的时候，会从`RedisSession`的cache的`MapSession`中将属性持久化到redis对应的set中。

- `redis`的[Redis Keyspace Notifications](https://redis.io/topics/notifications)，官网文档指出，此功能可以在redis的数据发生变化时，通过 Pub/Sub 将变动发布到对应的频道，可以用来监控数据的变化，例如

  - 订阅频道：`__keyspace@0__:mykey` 可以监控db0上的key为`mykey`的数据变化；
  - 订阅频道：`__keyspace@0__:del` 可以监控db0上的所有del命令操作的数据；

  如此一来便可以通过`Keyspace Notifications`订阅频道，获取session中属性的变化

### 解决方案

---

- 整体的思路就是监听spring-session对应的`redis`的db的`hset`、`hdel`操作，当检测到向session中设置值时，即可触发对应的类，并调用需要调用的方法即可。以监听`hset`为例，具体代码如下：

- 在配置文件中增加redis监听配置

  ```
  <!-- 定义Spring Redis的序列化器 -->
      <!-- String序列化 -->
      <bean id="stringRedisSerializer" class="org.springframework.data.redis.serializer.StringRedisSerializer"/>
  
      <!-- 将监听实现类注册到spring容器中 -->
      <bean id="dataSyncEventListener" class="com.yuyu.listener.LoginAttributeListener"/>
      <!-- 注册监听器并引入监听实现类 -->
      <bean id="messageListener" class="org.springframework.data.redis.listener.adapter.MessageListenerAdapter">
          <property name="delegate" ref="dataSyncEventListener"/>
          <property name="serializer" ref="stringRedisSerializer"/>
      </bean>
  
      <!-- 消息监听 -->
      <redis:listener-container>
          <!--指定消息处理方法，序列化方式及主题名称-->
          <redis:listener ref="messageListener" method="onMessage" serializer="stringRedisSerializer" topic="__keyevent@0__:hset"/>
  	</redis:listener-container> 
  ```

  

- 增加`MessageListener`的实现类

  ```
  public class SessionAttributeListener implements MessageListener {
  	// 注入session操作类
      @Resource(name = "sessionRepository")
      private RedisOperationsSessionRepository sessionRepository;
  
      /**
       * 监听redis的hset操作，并判断是否是向session中写入属性
       */
      @Override
      public void onMessage(Message message, byte[] pattern) {
          if (message == null || message.getChannel() == null 
          	|| message.getBody() == null) {
              return;
          }
          // 获取频道发布的消息
          String messageStr = new String(message.getBody());
          // 向session中设置属性了
          if(messageStr.startsWith("spring:session:sessions:")){
              // 获取sessionId
              String sessionId = messageStr.substring(messageStr.lastIndexOf(":") + 1);
              // 获取session
  			Session addAttributeSession = sessionRepository.getSession(sessionId);
          }
      }
  }
  ```

  