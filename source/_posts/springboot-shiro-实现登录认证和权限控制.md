---
title: springboot + shiro 实现登录认证和权限控制
date: 2018-05-18 15:47:35
categories: "Shiro" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - Shiro
     - Spring Boot
     - Mybatis-Plus
---
## 前言

这段时间在学习springboot，在spring security和shiro中选择了shiro，原因就是shiro学习成本比较低，可能没有Spring Security做的功能强大，但是在实际工作时可能并不需要那么复杂的东西,，而且粗粒度也可以根据需要来定制，所以使用小而简单的Shiro就足够了。本文主要参考了z77z的[SpringBoot+shiro整合学习之登录认证和权限控制](https://www.jianshu.com/p/672abf94a857)
[源码项目地址](https://github.com/a28520/shiro-example/tree/master/shiro-charter1)  

## 关于Shiro

**Shiro 的核心：**

* Subject(主体)： 用于记录当前的操作用户，Subject在shiro中是一个接口，接口中定义了很多认证授相关的方法，外部程序通过subject进行认证授权，而subject是通过SecurityManager安全管理器进行认证授权
* SecurityManager(安全管理器)：对Subject 进行管理，他是shiro的核心SecurityManager是一个接口，继承了Authenticator, Authorizer, SessionManager这三个接口。
* Authenticator(认证器)：对用户身份进行认证
* Authorizer(授权器)：用户通过认证后，来判断时候拥有该权限
* realm：获取用户权限数据
* sessionManager(会话管理)：shiro框架定义了一套会话管理，它不依赖web容器的session，所以shiro可以使用在非web应用上，也可以将分布式应用的会话集中在一点管理，此特性可使它实现单点登录。
* CacheManager(缓存管理器)：将用户权限数据存储在缓存，这样可以提高性能。

## 学习目标

对用户进行登录认证，若认证失败则返回至登录界面，只有认证成功且拥有权限才可以访问指定链接，否则跳转至403页面。

## 添加依赖

``` xml
<!-- shiro-->
<dependency>
	<groupId>org.apache.shiro</groupId>
	<artifactId>shiro-spring</artifactId>
	<version>1.4.0</version>
</dependency>	
```

## 添加shiro配置

配置中还添加了一个自定义的表单拦截器MyFormAuthenticationFilter，用来处理登录异常信息并通过返回

**ShiroConfig.java**

``` java
/**
 * @author wgc
 * @date 2018/02/09
 */
@Configuration
public class ShiroConfig {
    @Bean
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);

        //拦截器.
        Map<String,String> filterChainDefinitionMap = new LinkedHashMap<>();
        // 配置不会被拦截的链接 顺序判断

        filterChainDefinitionMap.put("/assets/**", "anon");
        filterChainDefinitionMap.put("/css/**", "anon");
        filterChainDefinitionMap.put("/js/**", "anon");
        filterChainDefinitionMap.put("/img/**", "anon");
        filterChainDefinitionMap.put("/layui/**", "anon");
        filterChainDefinitionMap.put("/captcha/**", "anon");
        filterChainDefinitionMap.put("/favicon.ico", "anon");

        //配置退出 过滤器,其中的具体的退出代码Shiro已经替我们实现了
        filterChainDefinitionMap.put("/logout", "logout");
        // <!-- 过滤链定义，从上向下顺序执行，一般将/**放在最为下边 -->:这是一个坑呢，一不小心代码就不好使了;
        // authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问;
        // user:认证通过或者记住了登录状态(remeberMe)则可以通过
        filterChainDefinitionMap.put("/**", "authc");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);

        // 如果不设置默认会自动寻找Web工程根目录下的"/login.jsp"页面
        shiroFilterFactoryBean.setLoginUrl("/login");
        // 登录成功后要跳转的链接
        shiroFilterFactoryBean.setSuccessUrl("/index");
        //未授权界面;
        shiroFilterFactoryBean.setUnauthorizedUrl("/403");
        
        //自定义拦截器
        Map<String, Filter> filters = shiroFilterFactoryBean.getFilters();
        filters.put("authc", new MyFormAuthenticationFilter());
        return shiroFilterFactoryBean;
    }

    /**
	 * 身份认证realm; (这个需要自己写，账号密码校验；权限等)
	 * @return myShiroRealm
	 */
    @Bean
    public MyShiroRealm myShiroRealm(){
        MyShiroRealm myShiroRealm = new MyShiroRealm();
        return myShiroRealm;
    }

    /**
     * 安全管理器
     * @return securityManager
     */
    @Bean
    public SecurityManager securityManager(){
        DefaultWebSecurityManager securityManager =  new DefaultWebSecurityManager();
        securityManager.setRealm(myShiroRealm());
        return securityManager;
    }
}
```

## Realm实现

Shiro从从Realm获取安全数据（如用户、角色、权限），就是说SecurityManager要验证用户身份，那么它需要从Realm获取相应的用户进行比较以确定用户身份是否合法；也需要从Realm得到用户相应的角色/权限进行验证用户是否能进行操作；可以把Realm看成DataSource，即安全数据源。Realm主要有两个方法：

* doGetAuthorizationInfo(获取授权信息)
* doGetAuthenticationInfo(获取身份验证相关信息)：

**自定义Realm实现**

``` java
/**
 * @author wgc
 * @date 2018/02/09
 */
public class MyShiroRealm extends AuthorizingRealm {
    private static final Logger logger = LoggerFactory.getLogger(MyShiroRealm.class);
    @Resource
    private ShiroService userInfoService;
    // 权限授权
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
        UserInfo userInfo  = (UserInfo)principals.getPrimaryPrincipal();
		for(SysRole sysRole : userInfoService.findSysRoleListByUsername(userInfo.getUsername())){
			authorizationInfo.addRole(sysRole.getRolename());
			logger.info(sysRole.toString());
			for(SysPermission sysPermission : userInfoService.findSysPermissionListByRoleId(sysRole.getId())){
				logger.info(sysPermission.toString());
				authorizationInfo.addStringPermission(sysPermission.getUrl());
			}
		};
        return authorizationInfo;
    }

    //主要是用来进行身份认证的，也就是说验证用户输入的账号和密码是否正确。
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token)
            throws AuthenticationException {
        //获取用户的输入的账号.
        String username = (String)token.getPrincipal();
        logger.info("对用户[{}]进行登录验证..验证开始",username);
        //通过username从数据库中查找 User对象，如果找到，没找到.
        //实际项目中，这里可以根据实际情况做缓存，如果不做，Shiro自己也是有时间间隔机制，2分钟内不会重复执行该方法
        UserInfo userInfo =  userInfoService.selectUserInfoByUsername(username);
        if(userInfo == null){
        	// 抛出 帐号找不到异常  
            throw new UnknownAccountException();  
        }        
        return new SimpleAuthenticationInfo(userInfo, userInfo.getPassword(), getName());
    }
}
```

## 自定义表单验证

在这里我们登录采用的是login页面post表单的方式，由于shiro已经内置了基于Form表单的身份验证过滤器FormAuthenticationFilter，如果不自定义会调用这个shiro默认的过滤器。因为需要返回登录失败信息，所以我们需要继承自定义一个表单过滤器，重写setFailureAttribute方法（当登录失败时会通过调用该方法），把登录失败信息写入到request的"shiroLoginFailure"属性中，由前端页面取出打印。

**表单认证过滤器实现**

``` java
/**
 * 自定义表单认证
 * @author wgc
 */
public class MyFormAuthenticationFilter extends FormAuthenticationFilter{
	private static final Logger logger = LoggerFactory.getLogger(MyFormAuthenticationFilter.class);
	/**
	 * 重写该方法, 判断返回登录信息
	 */
    @Override
    protected void setFailureAttribute(ServletRequest request, AuthenticationException ae) {
        String className = ae.getClass().getName();
        String message;
        String userName = getUsername(request);
        if (UnknownAccountException.class.getName().equals(className)) {
        	logger.info("对用户[{}]进行登录验证..验证未通过,未知账户", userName);
            message = "账户不存在";
        } else if (IncorrectCredentialsException.class.getName().equals(className)) {
        	logger.info("对用户[{}]进行登录验证..验证未通过,错误的凭证", userName);
            message = "密码不正确";
        } else if(LockedAccountException.class.getName().equals(className)) {
        	logger.info("对用户[{}]进行登录验证..验证未通过,账户已锁定", userName);
        	message = "账户已锁定";
        } else if(ExcessiveAttemptsException.class.getName().equals(className)) {
        	logger.info("对用户[{}]进行登录验证..验证未通过,错误次数过多", userName);
        	message = "用户名或密码错误次数过多，请十分钟后再试";
        } else if (AuthenticationException.class.getName().equals(className)) {
        	//通过处理Shiro的运行时AuthenticationException就可以控制用户登录失败或密码错误时的情景
        	logger.info("对用户[{}]进行登录验证..验证未通过,未知错误", userName);
        	message = "用户名或密码不正确";
        } else{
        	message = className;
        }
        request.setAttribute(getFailureKeyAttribute(), message);
    }
}
```

## web相关

### 登录页面

**controller**

``` java
@RequestMapping("/login")
public String loginForm() {
    return "login";
}
```

**login.html**

``` html
<!DOCTYPE html>
<html  lang="en" class="no-js" xmlns:th="http://www.thymeleaf.org">
<head>
	<meta charset="utf-8"/>
	<title>登录--layui后台管理模板</title>
	<link rel="stylesheet" href="../../layui/css/layui.css" media="all" />
	<link rel="stylesheet" href="../css/login.css" media="all" />
</head>
<body>
<div class="login">
	<h1>layuiCMS-管理登录</h1>
	<form class="layui-form" method="post">
		<div class="layui-form-item">
			<input class="layui-input" name="username" placeholder="用户名" type="text" autocomplete="off"/>
		</div>
		<div class="layui-form-item">
			<input class="layui-input" name="password" placeholder="密码" type="password" autocomplete="off"/>
		</div>
		<button class="layui-btn login_btn" lay-submit="" lay-filter="login">登录</button>
	</form>
</div>
<script type="text/javascript" src="../layui/layui.js"></script>

<script th:inline="javascript">
layui.use(['layer'], function(){
    var layer = layui.layer;
    var message = [[${shiroLoginFailure}]]?[[${shiroLoginFailure}]]:getUrlPara("shiroLoginFailure");
    if(message) {
    	layer.msg(message);
    }        	
});
function getUrlPara(name)
{
    var url = document.location.toString();
    var arrUrl = url.split("?"+name +"=");
    var para = arrUrl[1];
    if(para)
    	return decodeURI(para);
}
</script>
</body>
</html>
```

### 主页面

**controller**

``` java
@RequestMapping({"/","/index"})
public String index(Model model) {
    UserInfo user = (UserInfo)SecurityUtils.getSubject().getPrincipal();
    model.addAttribute("user", user);
    return "index";
}
```

**index.html**

``` html
<!DOCTYPE html>
<html lang="en" class="no-js" xmlns:th="http://www.thymeleaf.org">
<head>
  <meta charset="utf-8"/>
  <title>主页面</title>
</head>
<body >
<h3 th:text="${user.username}">user</h3>
</body>
</html>
```

## 测试

**任务一**
启动后，打开locahost:8080下任意链接由于没登录会跳转到login页面，登录成功后会进入index页面，点击主页面上的退出，则会注销登录并跳转至登录页面

**任务二**
登录之后访问add页面成功访问，在shiro配置文件中改变add的访问权限为

``` java
filterChainDefinitionMap.put("/add","perms[权限删除]");
```

再重新启动程序，登录后访问，会重定向到/403页面，由于没有编写403页面，报404错误。
上面这些操作，会触发权限认证方法：MyShiroRealm.doGetAuthorizationInfo()，每访问一次就会触发一次。