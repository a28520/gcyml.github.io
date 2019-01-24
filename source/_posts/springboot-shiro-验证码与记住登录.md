---
title: springboot-shiro-验证码与记住登录
date: 2018-05-21 16:41:06
categories: "Shiro" #文章分類目錄 可以省略
tags: #文章標籤 可以省略
     - Shiro
     - Spring Boot
     - Kaptcha
---
[源码项目地址](https://github.com/a28520/shiro-example/tree/master/shiro-charter2)

## 验证码实现

### 关于kaptcha

kaptcha 是一个很有用的验证码生成工具。有了它，你能够生成各种样式的验证码，由于它是可配置的。使用kaptcha能够方便的配置：

* 验证码的字体
* 验证码字体的大小
* 验证码字体的字体颜色
* 验证码内容的范围(数字，字母，中文汉字！)
* 验证码图片的大小。边框，边框粗细，边框颜色
* 验证码的干扰线(能够自己继承com.google.code.kaptcha.NoiseProducer写一个自己定义的干扰线)
* 验证码的样式(鱼眼样式、3D、普通模糊……当然也能够继承com.google.code.kaptcha.GimpyEngine自己定义样式)

### maven依赖

``` xml
<dependency>
	<groupId>com.github.penggle</groupId>
	<artifactId>kaptcha</artifactId>
	<version>2.3.2</version>
</dependency>		
```

### 注入验证码Servlet

**KaptchaConfig.java**

``` java
@Component
public class KaptchaConfig {
    @Bean
    public ServletRegistrationBean<KaptchaServlet> kaptchaServlet() {

        ServletRegistrationBean<KaptchaServlet> registrationBean = new ServletRegistrationBean<>(new KaptchaServlet(), "/captcha/kaptcha.jpg");

        registrationBean.addInitParameter(Constants.KAPTCHA_SESSION_CONFIG_KEY,
                Constants.KAPTCHA_SESSION_KEY);
        //宽度
        registrationBean.addInitParameter(Constants.KAPTCHA_IMAGE_WIDTH,"140");
        //高度
        registrationBean.addInitParameter(Constants.KAPTCHA_IMAGE_HEIGHT,"60");
        //字体大小
        registrationBean.addInitParameter(Constants.KAPTCHA_TEXTPRODUCER_FONT_SIZE,"50");
//        registrationBean.addInitParameter(Constants.KAPTCHA_BORDER_THICKNESS,"1"); //边框
        //无边框
        registrationBean.addInitParameter(Constants.KAPTCHA_BORDER,"no");
        //文字颜色
        registrationBean.addInitParameter(Constants.KAPTCHA_TEXTPRODUCER_FONT_COLOR, "blue");
        //长度
        registrationBean.addInitParameter(Constants.KAPTCHA_TEXTPRODUCER_CHAR_LENGTH, "4");
        //字符间距
        registrationBean.addInitParameter(Constants.KAPTCHA_TEXTPRODUCER_CHAR_SPACE, "6");

        //可以设置很多属性,具体看com.google.code.kaptcha.Constants
//      kaptcha.border  是否有边框  默认为true  我们可以自己设置yes，no
//      kaptcha.border.color   边框颜色   默认为Color.BLACK
//      kaptcha.border.thickness  边框粗细度  默认为1
//      kaptcha.producer.impl   验证码生成器  默认为DefaultKaptcha
//      kaptcha.textproducer.impl   验证码文本生成器  默认为DefaultTextCreator
//      kaptcha.textproducer.char.string   验证码文本字符内容范围  默认为abcde2345678gfynmnpwx
//      kaptcha.textproducer.char.length   验证码文本字符长度  默认为5
//      kaptcha.textproducer.font.names    验证码文本字体样式  默认为new Font("Arial", 1, fontSize), new Font("Courier", 1, fontSize)
//      kaptcha.textproducer.font.size   验证码文本字符大小  默认为40
//      kaptcha.textproducer.font.color  验证码文本字符颜色  默认为Color.BLACK
//      kaptcha.textproducer.char.space  验证码文本字符间距  默认为2
//      kaptcha.noise.impl    验证码噪点生成对象  默认为DefaultNoise
//      kaptcha.noise.color   验证码噪点颜色   默认为Color.BLACK
//      kaptcha.obscurificator.impl   验证码样式引擎  默认为WaterRipple
//      kaptcha.word.impl   验证码文本字符渲染   默认为DefaultWordRenderer
//      kaptcha.background.impl   验证码背景生成器   默认为DefaultBackground
//      kaptcha.background.clear.from   验证码背景颜色渐进   默认为Color.LIGHT_GRAY
//      kaptcha.background.clear.to   验证码背景颜色渐进   默认为Color.WHITE
//      kaptcha.image.width   验证码图片宽度  默认为200
//      kaptcha.image.height  验证码图片高度  默认为50
        return registrationBean;
    }
}
```

在这里我们注入了一个链接为“/captcha/kaptcha.jpg”的servlet。点击运行项目打开链接如果看到验证码图片，则说明配置成功了。

### 验证码拦截器

**CaptchaValidateFilter.java**

``` java
ublic class CaptchaValidateFilter extends AccessControlFilter {
    private String captchaParam = "captchaCode"; //前台提交的验证码参数名  
    
    private String failureKeyAttribute = "shiroLoginFailure";  //验证失败后存储到的属性名  
    
    public String getCaptchaCode(ServletRequest request) {
    	return WebUtils.getCleanParam(request, getCaptchaParam());
    }
    
	@Override
	protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue)
			throws Exception {
	    // 从session获取正确的验证码
	  	Session session = SecurityUtils.getSubject().getSession();
	    //页面输入的验证码
	    String captchaCode = getCaptchaCode(request);
	    String validateCode = (String)session.getAttribute(Constants.KAPTCHA_SESSION_KEY);
	    
	    HttpServletRequest httpServletRequest = WebUtils.toHttp(request);  
	    //判断验证码是否表单提交（允许访问）  
        if ( !"post".equalsIgnoreCase(httpServletRequest.getMethod())) {  
            return true;  
        } 
        
        // 若验证码为空或匹配失败则返回false
	    if(captchaCode == null) {
	    	return false;
	    } else if (validateCode != null) {
	    	captchaCode = captchaCode.toLowerCase();
	    	validateCode = validateCode.toLowerCase();
	        if(!captchaCode.equals(validateCode)) {
	        	return false;
	        }
	    }
	    return true;
	}

	@Override
	protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
		 //如果验证码失败了，存储失败key属性  
        request.setAttribute(failureKeyAttribute, "验证码错误");  
        return true;  
	}

	public String getCaptchaParam() {
		return captchaParam;
	}

	public void setCaptchaParam(String captchaParam) {
		this.captchaParam = captchaParam;
	}
}
```

验证码拦截器继承了AccessControlFilter，该类提供了访问控制的基础功能，比如是否允许访问/当访问拒绝时如何处理等。主要有两个方法：

* isAccessAllowed：表示是否允许访问；mappedValue就是[urls]配置中拦截器参数部分，如果允许访问返回true，否则false；
* onAccessDenied：表示当访问拒绝时是否已经处理了；如果返回true表示需要继续处理；如果返回false表示该拦截器实例已经处理了，将直接返回即可

### 修改ShiroConfig.java

``` java
public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);

        //拦截器.
        Map<String,String> filterChainDefinitionMap = new LinkedHashMap<>();
        // 配置不会被拦截的链接 顺序判断

        filterChainDefinitionMap.put("/css/**", "anon");
        filterChainDefinitionMap.put("/js/**", "anon");
        filterChainDefinitionMap.put("/img/**", "anon");
        filterChainDefinitionMap.put("/layui/**", "anon");
        filterChainDefinitionMap.put("/captcha/**", "anon");
        filterChainDefinitionMap.put("/favicon.ico", "anon");

        //配置退出 过滤器,其中的具体的退出代码Shiro已经替我们实现了
        filterChainDefinitionMap.put("/logout", "logout");
        //<!-- 过滤链定义，从上向下顺序执行，一般将/**放在最为下边 -->:这是一个坑呢，一不小心代码就不好使了;
        //<!-- authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问; user”表示访问该地址的用户是身份验证通过或RememberMe登录的都可以-->
        filterChainDefinitionMap.put("/add", "perms[add]");
        filterChainDefinitionMap.put("/login", "captchaVaildate,authc");

        filterChainDefinitionMap.put("/**", "user");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        
        // 如果不设置默认会自动寻找Web工程根目录下的"/login.jsp"页面
        shiroFilterFactoryBean.setLoginUrl("/login");
        // 登录成功后要跳转的链接
        shiroFilterFactoryBean.setSuccessUrl("/index");
        //未授权界面;
        shiroFilterFactoryBean.setUnauthorizedUrl("/403");

        //自定义拦截器
        Map<String, Filter> filters = shiroFilterFactoryBean.getFilters();
        filters.put("captchaVaildate", new CaptchaValidateFilter());
        filters.put("authc", new MyFormAuthenticationFilter());
        return shiroFilterFactoryBean;
    }
```

在表单验证拦截器前加入验证码拦截器

## 记住登录实现

### ShiroConfig的配置

在ShiroConfig.java中添加如下方法：

``` java
    public ShiroFilterFactoryBean shirFilter(SecurityManager securityManager) {
        ......
        shiroFilterFactoryBean.setSecurityManager(securityManager);
shiroFilterFactoryBean.setSecurityManager(securityManager);
        .....
}
  /**
     * 安全管理器
     * @return securityManager
     */
    @Bean
    public SecurityManager securityManager(){
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        securityManager.setRealm(myShiroRealm());
        
        //注入记住我管理器
        securityManager.setRememberMeManager(rememberMeManager());
        return securityManager;
    }
    
    /**
     * cookie对象;
     * rememberMeCookie()方法是设置Cookie的生成模版，比如cookie的name，cookie的有效时间等等。
     * @return rememberMeCookie
     */
    @Bean
    public SimpleCookie rememberMeCookie(){
        //这个参数是cookie的名称，对应前端的checkbox的name = rememberMe
        SimpleCookie simpleCookie = new SimpleCookie("rememberMe");
        //<!-- 记住我cookie生效时间30天 ,单位秒;-->
        simpleCookie.setMaxAge(30*24*60*60);
        simpleCookie.setHttpOnly(true);
        return simpleCookie;
    }
    
    /**
     * cookie管理对象;
     * rememberMeManager()方法是生成rememberMe管理器，而且要将这个rememberMe管理器设置到securityManager中
     * @return rememberMeManager
     */
    @Bean
    public CookieRememberMeManager rememberMeManager(){
        CookieRememberMeManager cookieRememberMeManager = new CookieRememberMeManager();
        cookieRememberMeManager.setCookie(rememberMeCookie());
        //rememberMe cookie加密的密钥 建议每个项目都不一样 默认AES算法 密钥长度(128 256 512 位)
        cookieRememberMeManager.setCipherKey(Base64.decode("3AvVhmFLUs0KTA3Kprsdag=="));
        return cookieRememberMeManager;
    }

......
```

## login页面

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
			<input class="layui-input" name="password" placeholder="密码"  type="password" autocomplete="off"/>
		</div>
		<div class="layui-form-item form_code">
			<input class="layui-input"  name="captchaCode" placeholder="验证码" lay-verify="required" type="text" autocomplete="off"/>
			<div>
				<img type="image" src="../captcha/kaptcha.jpg" id="codeImage" onclick="chageCode()" title="图片看不清？点击重新得到验证码" style="cursor:pointer;" width="116" height="36"/>
			</div>
		</div>
		<div class="layui-form-item">
			<input type="checkbox" name="rememberMe" title="记住我" lay-skin="primary"/>
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
    console.log(para);
    if(para)
    	return decodeURI(para);
}
function chageCode(){
    document.getElementById("codeImage").src="../captcha/kaptcha.jpg?"+Math.random();
}

</script>
</body>
</html>
```