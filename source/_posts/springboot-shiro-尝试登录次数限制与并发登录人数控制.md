---
title: springboot-shiro-尝试登录次数限制与并发登录人数控制
date: 2018-05-21 16:41:06
categories: "Shiro" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - Shiro
     - Spring Boot
     - Mybatis-Plus
---

[源码项目地址](https://github.com/a28520/shiro-example/tree/master/shiro-charter3)  

## 尝试登录次数控制实现

### 实现原理

Realm在验证用户身份的时候，要进行密码匹配。最简单的情况就是明文直接匹配，然后就是加密匹配，这里的匹配工作则就是交给CredentialsMatcher来完成的。我们在这里继承这个接口，自定义一个密码匹配器，缓存入键值对用户名以及匹配次数，若通过密码匹配，则删除该键值对，若不匹配则匹配次数自增。超过给定的次数限制则抛出错误。这里缓存用的是ehcache。

### shiro-ehcache配置

**maven依赖**

``` xml
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-ehcache</artifactId>
    <version>1.3.2</version>
</dependency>
```

**ehcache配置**

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<ehcache name="es">

    <diskStore path="java.io.tmpdir"/>

    <!--
       name:缓存名称。
       maxElementsInMemory:缓存最大数目
       maxElementsOnDisk：硬盘最大缓存个数。
       eternal:对象是否永久有效，一但设置了，timeout将不起作用。
       overflowToDisk:是否保存到磁盘，当系统当机时
       timeToIdleSeconds:设置对象在失效前的允许闲置时间（单位：秒）。仅当eternal=false对象不是永久有效时使用，可选属性，默认值是0，也就是可闲置时间无穷大。
       timeToLiveSeconds:设置对象在失效前允许存活时间（单位：秒）。最大时间介于创建时间和失效时间之间。仅当eternal=false对象不是永久有效时使用，默认是0.，也就是对象存活时间无穷大。
       diskPersistent：是否缓存虚拟机重启期数据 Whether the disk store persists between restarts of the Virtual Machine. The default value is false.
       diskSpoolBufferSizeMB：这个参数设置DiskStore（磁盘缓存）的缓存区大小。默认是30MB。每个Cache都应该有自己的一个缓冲区。
       diskExpiryThreadIntervalSeconds：磁盘失效线程运行时间间隔，默认是120秒。
       memoryStoreEvictionPolicy：当达到maxElementsInMemory限制时，Ehcache将会根据指定的策略去清理内存。默认策略是LRU（最近最少使用）。你可以设置为FIFO（先进先出）或是LFU（较少使用）。
        clearOnFlush：内存数量最大时是否清除。
         memoryStoreEvictionPolicy:
            Ehcache的三种清空策略;
            FIFO，first in first out，这个是大家最熟的，先进先出。
            LFU， Less Frequently Used，就是上面例子中使用的策略，直白一点就是讲一直以来最少被使用的。如上面所讲，缓存的元素有一个hit属性，hit值最小的将会被清出缓存。
            LRU，Least Recently Used，最近最少使用的，缓存的元素有一个时间戳，当缓存容量满了，而又需要腾出地方来缓存新的元素的时候，那么现有缓存元素中时间戳离当前时间最远的元素将被清出缓存。
    -->
    <defaultCache
            maxElementsInMemory="10000"
            eternal="false"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            overflowToDisk="false"
            diskPersistent="false"
            diskExpiryThreadIntervalSeconds="120"
    />

    <!-- 登录记录缓存锁定10分钟 -->
    <cache name="passwordRetryCache"
           maxEntriesLocalHeap="2000"
           eternal="false"
           timeToIdleSeconds="3600"
           timeToLiveSeconds="0"
           overflowToDisk="false"
           statistics="true">
    </cache>

</ehcache>
```

### #RetryLimitCredentialsMatcher

``` java
/** 
 * 验证器，增加了登录次数校验功能 
 * 此类不对密码加密
 * @author wgc
 */
@Component
public class RetryLimitCredentialsMatcher extends SimpleCredentialsMatcher {  
    private static final Logger log = LoggerFactory.getLogger(RetryLimitCredentialsMatcher.class);

    private int maxRetryNum = 5;
    private EhCacheManager shiroEhcacheManager;

    public void setMaxRetryNum(int maxRetryNum) {
        this.maxRetryNum = maxRetryNum;
    }

    public RetryLimitCredentialsMatcher(EhCacheManager shiroEhcacheManager) {
    	this.shiroEhcacheManager = shiroEhcacheManager; 
    }
	
  
    @Override  
    public boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) {  
    	Cache<String, AtomicInteger> passwordRetryCache = shiroEhcacheManager.getCache("passwordRetryCache");
        String username = (String) token.getPrincipal();  
        //retry count + 1  
        AtomicInteger retryCount = passwordRetryCache.get(username);  
        if (null == retryCount) {  
            retryCount = new AtomicInteger(0);
            passwordRetryCache.put(username, retryCount);  
        }
        if (retryCount.incrementAndGet() > maxRetryNum) {
        	log.warn("用户[{}]进行登录验证..失败验证超过{}次", username, maxRetryNum);
            throw new ExcessiveAttemptsException("username: " + username + " tried to login more than 5 times in period");  
        }  
        boolean matches = super.doCredentialsMatch(token, info);  
        if (matches) {  
            //clear retry data  
        	passwordRetryCache.remove(username);  
        }  
        return matches;  
    }  
} 
```

### Shiro配置修改

**注入CredentialsMatcher**

``` java
     /**
     * 缓存管理器
     * @return cacheManager
     */
    @Bean
    public EhCacheManager ehCacheManager(){
        EhCacheManager cacheManager = new EhCacheManager();
        cacheManager.setCacheManagerConfigFile("classpath:config/ehcache-shiro.xml");
        return cacheManager;
    }
     /**
     * 限制登录次数
     * @return 匹配器
     */
    @Bean
    public CredentialsMatcher retryLimitCredentialsMatcher() {
        RetryLimitCredentialsMatcher retryLimitCredentialsMatcher = new RetryLimitCredentialsMatcher(ehCacheManager());
        retryLimitCredentialsMatcher.setMaxRetryNum(5);
        return retryLimitCredentialsMatcher;

    }
```

 **realm添加认证器**

``` java
myShiroRealm.setCredentialsMatcher(retryLimitCredentialsMatcher());
```

## 并发在线人数控制实现

### KickoutSessionControlFilter

``` java
/**
 * 并发登录人数控制
 * @author wgc
 */
public class KickoutSessionControlFilter extends AccessControlFilter {

    private static final Logger logger = LoggerFactory.getLogger(KickoutSessionControlFilter.class);
    

    /**
     * 踢出后到的地址
     */
	private String kickoutUrl;

    /**
     * 踢出之前登录的/之后登录的用户 默认踢出之前登录的用户
     */
	private boolean kickoutAfter = false;
    /**
     * 同一个帐号最大会话数 默认1
     */
	private int maxSession = 1;

	private String kickoutAttrName = "kickout";
	
	private SessionManager sessionManager; 
	private Cache<String, Deque<Serializable>> cache; 
	
	public void setKickoutUrl(String kickoutUrl) { 
		this.kickoutUrl = kickoutUrl; 
	}
	
	public void setKickoutAfter(boolean kickoutAfter) { 
		this.kickoutAfter = kickoutAfter;
	}
	
	public void setMaxSession(int maxSession) { 
		this.maxSession = maxSession; 
	} 
	
	public void setSessionManager(SessionManager sessionManager) { 
		this.sessionManager = sessionManager; 
	}

	/**
	 * 	设置Cache的key的前缀
	 */
	public void setCacheManager(CacheManager cacheManager) { 
		this.cache = cacheManager.getCache("shiro-kickout-session");
	}
	
	@Override protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue)
			throws Exception {
		return false;
	} 
	
	@Override
	protected boolean onAccessDenied(ServletRequest request, ServletResponse response)
			throws Exception { 
		Subject subject = getSubject(request, response); 
		if(!subject.isAuthenticated() && !subject.isRemembered())
		{ 
			//如果没有登录，直接进行之后的流程 
			return true;
		} 
		
		Session session = subject.getSession();
		UserInfo user = (UserInfo) subject.getPrincipal(); 
		String username = user.getUsername();
		Serializable sessionId = session.getId();
		
		logger.info("进入KickoutControl, sessionId:{}", sessionId);
		//读取缓存 没有就存入 
		Deque<Serializable> deque = cache.get(username); 
		 if(deque == null) {
			 deque = new LinkedList<Serializable>();  
		     cache.put(username, deque);  
		 }  
		 
		//如果队列里没有此sessionId，且用户没有被踢出；放入队列
		if(!deque.contains(sessionId) && session.getAttribute(kickoutAttrName) == null) {
			//将sessionId存入队列 
			deque.push(sessionId); 
		} 
		logger.info("deque.size:{}",deque.size());
		//如果队列里的sessionId数超出最大会话数，开始踢人
		while(deque.size() > maxSession) { 
			Serializable kickoutSessionId = null; 
			if(kickoutAfter) { 
				//如果踢出后者 
				kickoutSessionId = deque.removeFirst(); 
			} else { 
				//否则踢出前者 
				kickoutSessionId = deque.removeLast(); 
			} 
			
			//踢出后再更新下缓存队列
			cache.put(username, deque); 
			try { 
				//获取被踢出的sessionId的session对象
				Session kickoutSession = sessionManager.getSession(new DefaultSessionKey(kickoutSessionId));
				if(kickoutSession != null) { 
					//设置会话的kickout属性表示踢出了 
					kickoutSession.setAttribute(kickoutAttrName, true);
				}
			} catch (Exception e) {
				logger.error(e.getMessage());
			} 
		} 
		//如果被踢出了，直接退出，重定向到踢出后的地址
		if (session.getAttribute(kickoutAttrName) != null && (Boolean)session.getAttribute(kickoutAttrName) == true) {
			//会话被踢出了 
			try { 
				//退出登录
				subject.logout(); 
			} catch (Exception e) { 
				logger.warn(e.getMessage());
				e.printStackTrace();
			}
			saveRequest(request); 
			//重定向	
			logger.info("用户登录人数超过限制, 重定向到{}", kickoutUrl);
			String reason = URLEncoder.encode("账户已超过登录人数限制", "UTF-8");
			String redirectUrl = kickoutUrl  + (kickoutUrl.contains("?") ? "&" : "?") + "shiroLoginFailure=" + reason;  
			WebUtils.issueRedirect(request, response, redirectUrl); 
			return false;
		} 
		return true; 
	} 
}
```
### ehcache配置
ehcache-shiro.xml加入
```
    <!-- 用户队列缓存10分钟 -->
    <cache name="shiro-kickout-session"
           maxEntriesLocalHeap="2000"
           eternal="false"
           timeToIdleSeconds="3600"
           timeToLiveSeconds="0"
           overflowToDisk="false"
           statistics="true">
    </cache>
```

### shiro配置

**ShiroConfig.java中注入相关对象**

``` java
     /**
     * 会话管理器
     * @return sessionManager
     */
    @Bean
    public DefaultWebSessionManager configWebSessionManager(){
        DefaultWebSessionManager manager = new DefaultWebSessionManager();
        // 加入缓存管理器
        manager.setCacheManager(ehCacheManager());
        // 删除过期的session
        manager.setDeleteInvalidSessions(true);

        // 设置全局session超时时间
        manager.setGlobalSessionTimeout(1 * 60 *1000);

        // 是否定时检查session
        manager.setSessionValidationSchedulerEnabled(true);
        manager.setSessionValidationScheduler(configSessionValidationScheduler());
        manager.setSessionIdUrlRewritingEnabled(false);
        manager.setSessionIdCookieEnabled(true);
        return manager;
    }

    /**
     * session会话验证调度器
     * @return session会话验证调度器
     */
    @Bean
    public ExecutorServiceSessionValidationScheduler configSessionValidationScheduler() {
    	ExecutorServiceSessionValidationScheduler sessionValidationScheduler = new ExecutorServiceSessionValidationScheduler();
    	//设置session的失效扫描间隔，单位为毫秒
    	sessionValidationScheduler.setInterval(300*1000);
    	return sessionValidationScheduler;
    }

    /**
     * 限制同一账号登录同时登录人数控制
     * @return 过滤器
     */
    @Bean
    public KickoutSessionControlFilter kickoutSessionControlFilter() {
        KickoutSessionControlFilter kickoutSessionControlFilter = new KickoutSessionControlFilter();
        //使用cacheManager获取相应的cache来缓存用户登录的会话；用于保存用户—会话之间的关系的；
        //这里我们还是用之前shiro使用的redisManager()实现的cacheManager()缓存管理
        //也可以重新另写一个，重新配置缓存时间之类的自定义缓存属性
        kickoutSessionControlFilter.setCacheManager(ehCacheManager());
        //用于根据会话ID，获取会话进行踢出操作的；
        kickoutSessionControlFilter.setSessionManager(configWebSessionManager());
        //是否踢出后来登录的，默认是false；即后者登录的用户踢出前者登录的用户；踢出顺序。
        kickoutSessionControlFilter.setKickoutAfter(false);
        //同一个用户最大的会话数，默认1；比如2的意思是同一个用户允许最多同时两个人登录；
        kickoutSessionControlFilter.setMaxSession(1);

        //被踢出后重定向到的地址；
        kickoutSessionControlFilter.setKickoutUrl("/login");
        return kickoutSessionControlFilter;
    }
```

**shiro过滤链中加入并发登录人数过滤器**

``` java
filterChainDefinitionMap.put("/**", "kickout,user");
```

访问任意链接均需要认证通过以及限制并发登录次数