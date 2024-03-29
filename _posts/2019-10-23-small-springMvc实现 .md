---
layout: post    
title: "不如写个small-springMvc"   
date: 2019-10-23    
description: "不如写个small-springMvc"
tag: spingMvc  
---

## WEB开发基础知识

### 基础知识

![img](https://pic1.zhimg.com/80/v2-1cb5b57ffab54838ad59983a8adbcdc8_hd.jpg)

一个**Http**请求到来，容器将其封装成servlet中的request对象，在request中可以获得所有的http信息，然后取出来操作。操作完成后再将其封装成servlet的response对象，应用容器将response对象解析后封装成一个**http response**。

**容器和servlet**

![img](https://pic2.zhimg.com/80/3fdb2abf692cb5edb833e139504ede39_hd.jpg)

**Tomcat=web服务器+servlet/jsp容器**

![img](https://pic4.zhimg.com/80/v2-c1761ba4e406196374fb7734966e8f97_hd.jpg)

![img](https://pic2.zhimg.com/80/v2-d8b75829a65958c65d50781155ae80a1_hd.jpg)

**三大组件（servlet，filter，listener）**

### SpringMVC和servlet的关系

![img](https://pic2.zhimg.com/80/v2-40ed984999cab23bc4e9e17a39e84839_hd.jpg)

Tomcat服务器启动时，创建1.ServletContext，此对象一直存活至关闭服务器。当http的request过来后，tomcat将其包装成2.HttpRequest类实例（也是域对象，单次请求共享），并创建空response实例，创建web.xml中对应的servlet实例，然后调用servlet中的service(request,response)，从servletContext中可以获取xml中存储的信息，所以不同的servlet间也可以贡献context域中内容。ServletContextListener监听context，HttpSessionListener监听3.HttpSession（同一会话有效，即多次请求有效）。最后返回response，这样一次完整的请求<—>响应流程就结束了。第4个域对象是Page，是jsp页面内共享数据。

![img](https://pic4.zhimg.com/v2-2530b17c1ee7e94bbcce7ca472d6a667_r.jpg)

在GenericServlet（即HttpServlet的直接父类）中，init方法将ServletConfig类对象由局部变量（Tomcat容器传给该servlet的init方法一个config）提升到成员变量，即所有继承自GenericServlet的子类servlet都自动具有config，而这个config持有对ServletContext的引用（servletContext有成员变量config，而config持有对servletCont引用）。所有域对象都持有对servletContext的引用。

**Filter**

![img](https://pic2.zhimg.com/80/v2-b8dfca0f5a4895bce75c2ce6b6f0c725_hd.jpg)

**映射器：**即<em>url->servlet</em>的映射关系,精准匹配，前缀匹配，扩展名匹配，如果都不匹配则交给DefaultServlet处理，这个类可以用来读取静态资源。

![img](https://pic1.zhimg.com/80/v2-42c3d43b3b7dd56851d1018d2186d1f0_hd.jpg)

DispatcherServlet，配置成.do，只拦截.do，但是起不到连接所有servlet的作用，而且有个.do很不优雅。配置成/，会导致静态资源也被拦截，且JSP也被拦截了。配置成/，此时拦截除jsp以外的所有，（因为JSPServlet会精确匹配.jsp结尾的，因此优先匹配），此时唯一要解决的就是静态资源。SpringMVC通过对静态资源目录下进行配置的方法<mvc;resources mapping="**" location="***">以判断某些资源是静态资源，不需要dispatchServlet处理，转而交给defaultServlet。

![img](https://pic1.zhimg.com/80/v2-e132fe10fc71bd79ce2d1f79964860d4_hd.jpg)

**所以SpringMVC就是一个DispatcherServlet么？不是，DispathcerServlet只是MVC的入口，他包含完整的组件。如下下章节所示：**

### SpringMVC与Spring的关系

> <https://blog.csdn.net/justloveyou_/article/details/74295728>

启动过程：Tomcat启动，生成ServletContext，因为在web.xml中配置了contextLoaderListener，因此当servletContext生成时，会被Spring监听到，然后spring初始化一个启动上下文，即ApplicationContext，其实现类是XmlWebApplication。

```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
        classpath:applicationContext.xml
    </param-value>
</context-param>
```

如上xml所配置的，ContextLoaderListener监听到servletContext后，从web.xml找到Spring的配置目录（通过context-param），即applicaitonContext.xml。这个xml将被封装成ApplicationContext，放到servletContext中，因此servletContext不仅包含web.xml内容，还包含applicationContext.xml内容的引用。

下一步开始初始化web.xml中的serlvet，一般必然存在DispathcerServlet，毕竟这个是MVC的入口servlet。而DispatcherServlet在初始化的时候，会先通过servletContext获取spring的ApplicationContext，然后再初始化自己的上下文。所以servlet间共享spring中的bean，而springMVC中的各个servlet又拥有自己独立的bean空间比如dispatherServlet的bean，就不会被其他servlet访问到。

**小结：tomcat启动–>servletContext–被监听–>Spring加载上下文–>SpringMVC获取Spring上下文并为每个servlet创建上下文，尤其是dispathcerServlet（这个尤其是因为我的项目里只配置了dispatcherServlet）。**

+ Spring容器与SpringMVC容器的联系与区别
  * SpringMVC因为持有spring的context，因此getBean时先从自己的上下文中获取，如果没有，则向Spring获取。而Spring不持有MVC的，因此无法获取子容器的bean。一般项目中，service、dao包中实例交给Spring创建，而controller包的实例交给MVC创建。
  * 子容器的创建后于父容器，当Spring上下文创建完毕后，才轮到mvc的创建。

### SpringMVC运行流程

1. HandlerMapping接口实现将请求映射到应该处理它的类或者方法。
2. HandlerAdapter接口实现用特定的模式处理请求，如常规servlet，更复杂的MVC工作流或者POJO bean中的方法。
3. ViewResolver接口允许使用不同的模板引擎，即视图解析器。
4. 使用Apache Commons文件上传或者变现自己的MultipartResolver解析multipart请求。
5. 使用LocaleResolver解决语言环境问题。

**流程小结：请求—>DispatcherServlet—>HandlerMapping–获得ExecutionChain（拦截器+handler）---->handlerAdapter---->返回modelAndView—>ViewResolver–>view---->渲染----->返回响应**

## 第一步：初始化

### 需求

+ **Spring的初始化：**启动Tomcat时，Spring容器监听到ServletContext后，读取web.xml中context-param配置项，获取Spring的applicationContext.xml的地址，然后解析xml并完成xml中所有bean的创建。
+ **SpringMVc的初始化：**调动dispatcherServlet时，SpringMVC读取web.xml中init-param配置项，定位到其xml配置文件，解析后完成所有bean的创建。

### 实现

#### Spring的初始化

1. 创建类ContextLoaderListener impelements ServletContextListener可以监听tomcat的启动，注意这一步也要在web.xml中配置listener标签，这样tomcat才能感知到有这个listener。

   ```xml
   <listener>
       <listener-class>com.sonihrmvc.framework.ContextListener.ContextLoaderListener</listener-class>
   </listener>
   ```

2. 此时通过listener已经能够感知tomcat何时启动，那我们就要准备初始化Spring容器了。首先在WEB-INF下建立lib包，lib包中导入我们之前的sonihr-Spring项目打包成的jar包（这一步不会的可以百度，maven项目的打包其实只要点一下就行，然后在新项目的pom里要导入原来sonihr-Spring项目的依赖），如果你的tomcat启动时提示NoClassDefFoundError异常，那大概率是你的jar包没有装好。。下图为项目当前的结构
   ![img](http://img.sonihr.com/bfaefb7f-71a2-4737-861b-d7caa385ae23.jpg)

3. 装好jar包后，填充listener的逻辑部分,本质上就是调用classPathXmlApplicationContext（这是sonihr-spring中的，这个类用于从xml或注解中进行实例的创建），读取从web.xml的context-param中的contextConfigLocation值，即spring的配置文件的地址，然后创建实例。至此，Spring的初始化就完毕了。**如果上文有不懂的，我分析一下：1.不知道web.xml是干嘛用的，不知道web.xml应该配置什么，listener标签，context-param标签不知道是啥。2.不会maven，不知道web项目要把jar包放在WEB-INF/lib下并设置为库文件。3.没看过sonihr-spring教程，不懂怎么从xml中生成实例。如果这几步不很清楚，那建议夯实基础。**

   ```java
   public class ContextLoaderListener implements ServletContextListener {
       @Override
       public void contextInitialized(ServletContextEvent sce) {
           ServletContext servletContext = sce.getServletContext();
           String springXmlPath = servletContext.getInitParameter("contextConfigLocation");
           if(springXmlPath.startsWith("classpath:")){
               springXmlPath = springXmlPath.substring(10);
           }
           ApplicationContext applicationContext = null;
           try {
               applicationContext = new ClassPathXmlApplicationContext(springXmlPath);
               Person person = (Person) applicationContext.getBean("person");
               System.out.println(person);
           } catch (Exception e) {
               e.printStackTrace();
           }
       }
   }
   ```

4. 这边做了一个substring，是因为配置的是classpath:applicationContext.xml，实际上applicationContext.xml我就是放置在resources中，即根据target文件夹，就是WEB-INF/classes下，所以可以不用写。（代码极不健壮）

   ```xml
   <context-param>
       <param-name>contextConfigLocation</param-name>
       <param-value>classpath:applicationContext.xml</param-value>
   </context-param>
   ```

5. 做个简单测试，创建一个Person类，注解用@Service，然后private String name的参数用@Value(“”)注解，因为sonihr-spring已经实现了注解注入，因此tomcat启动时，创建ServletContext时，被ContextLoaderListener监听到，然后对Spring容器进行初始化。测试时，打印person属性，发现注入成功。

   ![img](http://img.sonihr.com/cccc68bf-cf6c-4d4f-8d5a-dae55b61194a.jpg)

#### SpringMVC的初始化

1. 创建<em>dispatherSerlvet extends HttpServlet</em>，并且在xml中注册。在当前版本中，我们还只拦截.do结尾了，为了防止静态资源也被拦截。注意在配置中的<em>init-param</em>，在servlet的生命周期中，可以重写servlet的无参init方法，在init方法中对<em>SpringMVC</em>进行初始化。

   ```xml
   <servlet>
       <servlet-name>dispatcherServlet</servlet-name>
       <servlet-class>com.sonihrmvc.framework.servlet.DispatcherServlet</servlet-class>
       <init-param>
       <param-name>contextConfigLocation</param-name>
       <param-value>classpath:mvcContext.xml</param-value>
       </init-param>
       <load-on-startup>1</load-on-startup>
   </servlet>
   <servlet-mapping>
       <servlet-name>dispatcherServlet</servlet-name>
       <url-pattern>*.do</url-pattern>
   </servlet-mapping>
   ```

2. 我们要实现以下需求：**1.tomcat的ServletContext持有Spring的springContext，springMVC同时持有spring和自己的context。2.springMVC在getBean时，如果在自己的context获取不到，则去获取spring的context中的bean。**第1个需求简单，只要在ServletContext中setAttribute即可。但是第2个呢？我们需要改动sonihr-spring项目了。因为在sonihr-spring项目中，applicationContext是不存在父子容器这种说法的，因此我们需要在AbstractApplicationContext模板类中加入ApplicationContext类变量parent，这个用于指向父容器。但是创建bean和获取bean都是在beanfactory中进行的，因此beanFactory中要设置ApplcationContext变量，用于保存当前调用beanFactory的ApplicationContext实例。在getbean时，优先检查当前context中是否有该beanName，如果没有则从低向高搜索父context中是否存在。**sonihr-spring的改动在tag：v1.6-solve-parentContext。**

   ```java
    public Object getBean(String name) throws Exception {
           BeanDefinition beanDefinition = beanDefinitionMap.get(name);
           ApplicationContext context = this.getContext();
           while (beanDefinition==null&&context.getParent()!=null){
               ApplicationContext parent = context.getParent();
               Object object = parent.getBean(name);
               if(object!=null){
                   return object;
               }else{
                   context = parent;
               }
           }
           if(beanDefinition==null)
               throw new IllegalArgumentException("No bean named " + name + " is defined");
           Object bean = beanDefinition.getBean();
           //如果bean==null说明还未存在，不是单例说明是否存在都要重新创建
           if(bean==null||!beanDefinition.isSingleton()){
               bean=doCreateBean(name,beanDefinition);//根据生命周期来的，先创建后进行before，init,after
               bean = initializeBean(bean,name);//
               beanDefinition.setBean(bean);
           }
           return bean;
       }
   ```

3. 在DispatcherServlet的init方法中对SpringMVC进行初始化。这边要注意，我在ClassPathXmlApplicationContext构造方法中就指定了父类，因为如果你必须在读取xml创建bean前就已经设置好parent属性。

   ```java
   public class DispatcherServlet extends HttpServlet {
       @Override
       public void init() throws ServletException {
           super.init();
           String mvcXmlPath = this.getInitParameter("contextConfigLocation");
           if(mvcXmlPath==null||mvcXmlPath.length()==0)
               return;
           if(mvcXmlPath.startsWith("classpath:")){
               mvcXmlPath = mvcXmlPath.substring(10);
           }
           ServletContext servletContext = this.getServletContext();
           ApplicationContext mvcContext = null;
           try {
               ApplicationContext springContext = (ApplicationContext)servletContext.getAttribute("springContext");
               mvcContext = new ClassPathXmlApplicationContext(springContext,mvcXmlPath);
               System.out.println(springContext);
               PersonService personService = (PersonService) mvcContext.getBean("personService");
               System.out.println("personService=" + personService);
               PersonController personController = (PersonController) mvcContext.getBean("personController");
               System.out.println("personController=" + personController);
           } catch (Exception e) {
               e.printStackTrace();
           }
       }
   }
   ```

   打印结果：

   ![img](http://img.sonihr.com/d66fe2bd-1b2d-4ac8-aecf-ff2febbb7e7e.jpg)

4. 小结：**目前为止，已经完成了Service包用@Service注解，并通过Spring的context创建实例，Controller包用@Controller注解，通过springmvc初始化创建实例。controller依赖service，通过@Autowired进行注入。tomcat启动后，spring和springMVC的实例全部创建完毕。**

## 第二步：HandlerMapping

### 需求

![img](http://img.sonihr.com/5a20c50e-fff5-4eee-825d-676327c0ce75.jpg)

1. 对于上图，当输入正确URL时，控制台可以正确打印I am eating。即，可以通过@RequestMapping的方式，解析URL，定位到eating方法并执行。
2. 利用AOP实现拦截器链。

### @RequestMapping的实现

![img](http://img.sonihr.com/99c2b65f-8027-403d-a936-fa1cf1442bba.jpg)

1. 目录结构如上图所示

   <em>RequestMapping</em>是注解，目前只有一个变量就是value。HandlerExecutionChain是HandlerMapping组件返回给dispatcherServlet的返回值，其中封装了1个RequestMappingHandler实例和1个拦截器列表（目前还未实现）。RequestMappingHandler中封装了要执行的方法method，执行方法的对象bean以及参数args。AnnotationHandlerMapping是最重要的，其中保存了一个HashMap<String,RequestMappingHandler> handlerRegistry，其中注册了controller包中所有被@RequestMapping解析后的url与对应方法的键值对。

2. 首先在dispatcherServlet的init方法中进行初始化，service方法中进行doDispatcher。doDispatcher目前用于将请求的url传递给AnnotationHandlerMapping，然后返回相匹配的RequestMappingHandler和拦截器们。

   ```java
   private void doDispatch(HttpServletRequest request,HttpServletResponse response) throws Exception {
       //Todo：
       HandlerExecutionChain handlerExecutionChain =  handlerMapping.getHandler(request);
       RequestMappingHandler handler = handlerExecutionChain.getHandler();
   
       //至于如何传参，就是HandlerAdapter的事情了
       handler.getMethod().invoke(handler.getBean(),null);//和AOP不冲突，内部bean如果是代理类，会调用代理后方法,
   }
   ```

3. AnnotationHandlerMapping通过遍历beanFactory中的beanDefinitionMap，获得了@RequestMapping注解内的值，从而解析出request请求url所对应的方法是谁。

   ```java
   @Override
   public void init() {
       AbstractBeanFactory beanFactory = mvcContext.getBeanFactory();
       Map<String, BeanDefinition> map = beanFactory.getBeanDefinitionMap();
       for(Map.Entry<String, BeanDefinition> entry:map.entrySet()){
           String prefix = null;
           String suffix = null;
           Class clazz = entry.getValue().getBeanClass();//通过类名获得前缀
           Object bean = entry.getValue().getBean();
           Annotation annotation = clazz.getAnnotation(RequestMapping.class);
           if(annotation!=null){
               prefix = ((RequestMapping)annotation).value();
           }else{
               continue;
           }
           Method[] methods = clazz.getMethods();//通过方法获得后缀
           for(Method method:methods){
               annotation =method.getAnnotation(RequestMapping.class);
               if(annotation!=null){
                   suffix = ((RequestMapping)annotation).value();
                   String url = prefix + suffix;
                   handlerRegistry.put(url,new RequestMappingHandler(bean,method,null));
                   //System.out.println("url = "+url);
               }else{
                   continue;
               }
           }
       }
   }
   
   @Override
   public HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   
       HandlerExecutionChain handlerExecutionChain = new HandlerExecutionChain();
       //System.out.println("uri = "+request.getRequestURI());
       RequestMappingHandler handler = handlerRegistry.get(request.getRequestURI());
       //System.out.println("handler = "+handler);
       handlerExecutionChain.setHandler(handler);
       return handlerExecutionChain;
   }
   ```

### 拦截器链的实现

1. 解决一个纠结了一下午+一晚上的问题。Spring中的拦截器是基于AOP的，但是SpringMVC中HandlerInteceptor却不是基于AOP，而是基于职责链。所以AOP也可以实现拦截器，NVC的HandlerInterceptor也可实现拦截器。

2. HandlerInterceptor接口的实现类均为MVC的拦截器类，这个接口规定了四个方法。实际的MVC中没有getpath方法，我放在这个就不需要xml配置了，直接在方法里规定要被拦截的地址即可。

   ```java
   public interface HandlerInterceptor{
       String[] getPath();//该方法规定被拦截的地址
       //该方法在请求处理之前调用，返回true表示交给下一个拦截器，返回false表示到此为止
       boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
       //视图返回之后，渲染之前被调用
       void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception;
       void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception;
   }
   ```

3. 说一下MVC中拦截器的方法调用顺序。假设有A,B,C,D拦截器，请求过来后会调用A.pre->B.pre->C.pre->D.pre->交给适配器处理请求,返回modelAndView->D.post->C.post->B.post->A.post->渲染视图->D.after->C.after->B.after->A.after。即pre是顺序的，post和after都是逆序的。不仅如此，如果C的pre返回false，顺序是A.pre->B.pre->c.pre(false)->B.after->A.after，不会走post也不会处理请求。
   ![img](https://oscimg.oschina.net/oscnet/a255c6a82ec3e110205bf8cbbf546dcf32b.jpg)

4. 我们需要在AnnotationHandlerMapping中获取所有HandlerInterceptor的实现类实例，放入list中，然后判断是否和当前请求的uri匹配，将所有匹配的拦截器加入list后，利用setter方法加入HandlerExecutionChain。

5. 在DispathceServlet中获得的HandlerExecutionChain中拥有了针对当前uri的拦截器组和handler。在handler传递给HandlerAdapter组件之前，先调用拦截器组的pre方法，如果pre方法中有返回false，即反向调用after。

   ```java
   private void doDispatch(HttpServletRequest request,HttpServletResponse response) throws Exception {
       HandlerExecutionChain handlerExecutionChain =  handlerMapping.getHandler(request);
       List<HandlerInterceptor> handlerInterceptors = handlerExecutionChain.getInterceptors();
       RequestMappingHandler handler = handlerExecutionChain.getHandler();
       for(int i=0;i<handlerInterceptors.size();i++){
           HandlerInterceptor interceptor = handlerInterceptors.get(i);
           if(!interceptor.preHandle(request,response,handler)){
               for(int j=i-1;j>=0;j--){
                   handlerInterceptors.get(j).afterCompletion(request,response,handler,new Exception());
               }
               break;
           }
       }
       //至于如何传参，就是HandlerAdapter的事情了
       handler.getMethod().invoke(handler.getBean(),null);//和AOP不冲突，内部bean如果是代理类，会调用代理后方法,
   }
   ```

   **小结：第二步后，请求已经可以正确的派发到Controller的方法上，但是方法参数之类的还未能传递，叫交给HandlerAdapter组件。**

## 第三步：HandlerAdapter

### HandlerMapping和HandlerAdapter的区别

+ 对于项目中只用到注解方式的我来说，一直心中有一个疑问：既然已经可以通过HandlerMapping映射到具体方法了，那直接反射调用方法不就完了么？为什么还要多一个HandlerAdapter组件呢？因为Spring框架是慢慢发展过来的，要保证对之前的兼容性，同时还要保证扩展性，因此HandlerMapping着重于对类的匹配（早期的handler即为类，而不是具体方法，在Spring4以后handler也可以是具体方法），HandlerAdapter着重于对具体方法的调用。
+ 由上图看出，HandlerMapping的实现类有两个分支，一个是HandlerMethodMapping，一个是UrlHandlerMapping，前者的实现类是RequestMappingHandlerMapping，其实就是我们常用的@RequestMapping注解，使用这个注解实现requestMapping接口功能时，可以直接匹配到相关的方法。其他的，比如实现Controller接口的ControllerClassNameHandlerMapping或者xml文件中配置的sampleUrlHandlerMapping或者利用bean名称的BeanNameUrlHandlerMapping都是匹配到相关的实现类。举例来说，Controller的实现类中必须实现handleRequest方法，因此在只需要匹配到这个实现类，然后在Adapter中反射调用handleRequest方法即可。
+ **小结：HandlerMapping将URL映射为方法或者类，然后交给HandlerAdapter进一步处理，这个处理主要是调用对应的方法，填充参数，最后返回ModelAndView给DispatcherServlet。**

### 对第二步的优化

![img](http://img.sonihr.com/3161d625-f017-462b-82be-60a9549f9376.jpg)

+ 第二步中，我们只考虑了@RequestMapping注释这一种方式，即我们的AnnotationHandlerMapping类。我们抽象出一个AbstractHandlerMapping implements HandlerMapping。这个抽象类为模板类，子类可以去重写他的registryURLAndHandler方法。因为在AnnotationHandlerMapping中，这个方法就是解析注解，新建的ControllerHandlerMapping中就是获得Controller接口实现类的bean名称和实现类实例，新建的BeanNameHandlerMapping中根据bean的名称和bean的实例注册，新建的SimpleUrlHandlerMapping中就是解析xml，从而进一步注册。
+ 在DisptcherServlet中重构代码，在doInit方法中初始化所有HandlerMapping，这样就可以获得所有的url和handler的对应关系。要注意，AbstractHandlerMapping中的map和拦截器组list需要设计为static，以防止每次子类实例init的时候，都加入同一个map和拦截器组list中。

### 需求

1. 实现适配器设计模式，HandlerAdapter接口。
2. 支持多种适配方式，比如继承Controller接口的handler，普通servlet作为handler。但是本项目着重在于@RequestMapping注解下，类型为RequestMappingHandler的handler。
3. 实现参数传递，任意类型，包括对象。

### 适配器设计模式的实现

1. 设计HandlerAdapter接口，具有supports和handle两个方法。前者作用为实现职责链设计模式，在DispatcherServlet中，遍历所有的HandlerAdapter方法，如果supports返回ture，即采用当前adapter，否则交给下一个。后者作用是包装处理方法，所有adapter的处理方法都被handle包装，这样用户不需要知道内部的实现细节，dispatcherServlet只要先遍历找到supports返回true的adapter，然后执行adapter的handle方法即可。

   ```java
   public interface HandlerAdapter {
       boolean supports(Object handler);
       ModelAndView handle(HttpServletRequest request,HttpServletResponse response,Object handler) throws Exception;
   }
   ```

### 参数匹配

1. 先处理AnnotationHandlerAdapter类的handle方法：

   ```java
   @Override
   public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
       RequestMappingHandler rmHandler = (RequestMappingHandler)handler;
       Object[] args = ArgumentResolverUtil.resloveRequsetParam(request,rmHandler.getMethod());
       Object obj = rmHandler.getMethod().invoke(rmHandler.getBean(),args);
       return new ModelAndView(obj);
   }
   ```

   重点全在ArgumentResolverUtil这个工具类。为什么要拆出一个工具类呢？因为众多的handle中必然都需要解析request传过来的参数，因此单独做一个工具类可以避免耦合。

2. 因此在逻辑判断中，首先判断是否在argMap中存在相同的名字：比如方法是void speak(int age)，如果request自带一个参数为age，那么就刚好匹配。如果不同，不如方法是void baby(String name,Baby baby,int age),但是request传来的传输是name=xxx,babyName=yyy,babyAge=1,weight=10,age=33，此时发现中间那一段是baby的属性，因此babyName，babyAge，weight这三个虽然出现在了argMap中，但是却不是baby这个方法的形参名字，要通过BeanUtils将这三个参数组合成Baby的实例，然后通过cast动态转型转换成parameter需要的类型。

3. 因此在逻辑判断中，首先判断是否在argMap中存在相同的名字：比如方法是void speak(int age)，如果request自带一个参数为age，那么就刚好匹配。如果不同，不如方法是void baby(String name,Baby baby,int age),但是request传来的传输是name=xxx,babyName=yyy,babyAge=1,weight=10,age=33，此时发现中间那一段是baby的属性，因此babyName，babyAge，weight这三个虽然出现在了argMap中，但是却不是baby这个方法的形参名字，要通过BeanUtils将这三个参数组合成Baby的实例，然后通过cast动态转型转换成parameter需要的类型。

   ```java
   public static Object[] resloveRequsetParam(HttpServletRequest request, Method method) throws Exception {
       Map<String,String[]> paramMap = request.getParameterMap();
       Map<String,String> argMap = new LinkedHashMap<>();
       for(Map.Entry<String,String[]> entry:paramMap.entrySet()){
           String paramName = entry.getKey();
           String paramValue = "";
           String[] paramValueArr = entry.getValue();
           for(int i=0;i<paramValueArr.length;i++){
               if(i==paramValueArr.length-1)
                   paramValue += paramValueArr[i];
               else
                   paramValue += paramValueArr[i] + ",";
           }
           argMap.put(paramName,paramValue);//处理后的request键值对
       }
   
       Parameter[] parameters = method.getParameters();
       Object[] args = new Object[parameters.length];
       for(int i=0;i<parameters.length;i++){
           Parameter parameter = parameters[i];
           if(argMap.containsKey(parameter.getName())){
               String value = argMap.get(parameter.getName());
               Type type = parameter.getType();
               if(type == String.class)
                   args[i] = value;
               else
                   args[i] = ConverterFactory.getConverterMap().get(parameter.getType()).parse(value);
           }else {
               Type type = parameter.getType();
               Object bean = ((Class) type).newInstance();
               try{
                   BeanUtils.populate(bean,argMap);
                   args[i] = ((Class) type).cast(bean) ;
               }catch(Exception e){
                   args[i] = null;
               }
           }
       }
       return args;
   }
   ```

4. 还有一个疑问。我凭什么能获取到方法参数的形参名称？即步骤三中的name，baby，age。这是java1.8的新特性，但是默认是不可以的，因为会增加class文件的大小。怎么开启呢？以IDEA为例，注意红圈处

   ![img](http://img.sonihr.com/da23c74d-37b3-4714-987f-2308afd09358.jpg)

   **小结：本步骤实现了适配器设计模式，并且能将request请求携带的参数正确传递给相应的方法并调用。值得注意的是，返回值是ModelAndView，也是下一步我们要处理的。**

## 第四步：ViewResolver和View

### ViewResolver和View的关系

1. 对于控制器的目标方法，无论其返回值是String，View，ModelMap或是ModelAndView，SringMVC都会在内部将其封装为一个叫做ModelAndView的对象返回。这个ModelAndView会经过视图解析器（ViewResolver）解析成为最终的视图对象。
2. 即，你控制器目标方法返回同一个字符串，会根据视图解析器的不同，生成不同的视图对象View。
3. 视图对象View会调用render方法对视图进行渲染，得到response结果。

### 需求

1. 通过Model实现对视图的传参
2. 实现JSP，HTML的视图展示，支持转发和重定向。
3. 实现@ResponseBody注解功能。

### Model的实现

1. 为什么要有Model？为什么不干脆放到Request域中或者session域中呢？因为JSP技术汇总用到了Request等域对象，但是如果前端不是JSP技术呢？通过Model来传参，实现了不依赖于任何前端技术，如果前端是jsp，那我把model的参数交给域对象即可，如果前端是别的，那我就交给那个前端可以识别的数据。实现了解耦，这真是MVC中的一步好棋。

2. 本项目根据需求，只用实现JSP的传参即可，因此Model最后要交给Request域。Model是LinkedHashMap<String,Object>类型的。用法是，对于下列控制器的目标方法：

   ```java
   public String list(Model model) {
           //获取列表页
           List<Seckill> list = seckillService.getSeckillList();
           model.addAttribute("list", list);
           //list.jsp + model = ModelAndView
           return "list";// /WEB-INF/jsp/"list".jsp
   }
   ```

   只要参数列表中有Model形参，那么默认返回值ModelAndView中的model就是这个。

3. 因此要修改ArgumentResolverUtil工具类，增加对Model类型形参的识别。下面这段代码中值得注意的是，在Util的方法中传入的model，这是因为如果控制器目标方法参数中有Model类型的形参，那么要提前创建这个Model对象出来，这样在反射调用方法后，model中就有了值，简单来说就是通过提前new Model()作为参数，在调用方法后，model的改变可以被感知到。因为java是址传递，因此传入反射方法中的参数是拷贝了一份引用传递进去，但是拷贝的引用和原来的引用指向堆中同一块内存区域，在方法中利用引用修改被引用内存区域中的值时，方法外的引用也能感知到。因此，在反射调用控制器目标方法后，model会感知到方法内对model参数进行的改变。

   ```java
   @Override
   public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
       RequestMappingHandler rmHandler = (RequestMappingHandler)handler;
       Model model = new Model();
       Object[] args = ArgumentResolverUtil.resloveRequsetParam(request,rmHandler.getMethod(),model);
       Object obj = rmHandler.getMethod().invoke(rmHandler.getBean(),args);
       if(obj instanceof ModelAndView)
           return (ModelAndView)obj;
       ModelAndView mv = new ModelAndView();
       if(obj instanceof String){
           mv.setModel(model);
           mv.setView(obj);
       }
       return mv;
   }
   
   ArgumentResolverUtil类中：
   ...
   Parameter parameter = parameters[i];
   //如果形参中有Model类，则创建一个参数
   if(parameter.getType() == Model.class){
       args[i] = model;
       continue;
   }
   if(argMap.containsKey(parameter.getName())){
   ...
   ```

4. 这一步做完后，对于任意的控制器目标方法**返回值 方法名(Model model,其他参数)**，只要你在方法内部用model.put(String,Object)，即可传递数据保存至model。

### ViewReslover和View的实现

1. View接口，内部一个render方法，用于将view对象返回给浏览器。ViewResolver接口，内部一个resolveViewName方法，用于将viewName解析成View实例。分别建立实现类为InternalResouceView和InternalResourceViewResolver。

2. 当DispatcherServlet获得ModelAndView后，通过ViewResolve将viewName转化成View实例，然后将view实例调用render方法将结果返回给浏览器。

   ```java
   //视图解析器解析mv
   View view = resolver.resolveViewName(mv.getView());
   //页面渲染
   view.render(mv.getModel(),request,response);
   ```

3. 一个页面解析器可以解析不止一种类型的页面。正如在mvcContext.xml中配置的那样，不仅要配置视图解析器，还要配置视图解析器内视图的类型。我们这里解析的是JSP，分重定向或者转发两种情况即可。

   ```java
   @Override
   public View resolveViewName(String viewName) throws Exception {
       if(viewClass.equals("com.sonihrmvc.framework.view.InternalResourceView")){
           if(viewName.startsWith("redirect:"))
               return new InternalResourceView(viewName.substring(9),true);
           else
               return new InternalResourceView(prefix + viewName + suffix,false);
       }
       return null;
   }
   ```

   这里其实运用的是静态工厂设计模式，会出现大量的if…else…。但是因为ResolveView中已经有了view的className，其实可以用反射实现动态工厂。通过反射创建实例，然后为path动态赋值。

4. 重定向和转发的时候要注意路径。重定向是浏览器做的，因此配置路径时，/表示主机名：即localhost:8080/，转发是服务器做的，因此配置路径时，/表示应用名：即localhost:8080/mvc/。也很好理解，因为浏览器不知道主机有多少应用，因此默认在主机根目录，服务器知道当前在运行哪一个应用，因此默认在当前应用根目录。相对路径指的是当前路径父路径+新增路径。

5. 在render方法中就是将model中舒服传递给request域对象，然后重定向或转发到特定的jsp。

   ```java
   @Override
   public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
       for(Map.Entry<String,?> entry:model.entrySet()){
           request.setAttribute(entry.getKey(),entry.getValue());
       }
       if(!this.IsRedirect)
           request.getRequestDispatcher(path).forward(request,response);
       else
           response.sendRedirect(path);
   }
   ```

### @ResponseBody的实现

1. 设计一个@ResponseBody注解，不需要有任何值，只是标注即可。

2. 在AnnotationHandlerAdapter的handle方法中，判断目标方法是否具有ResponseBody注解，如果有则直接response对象向浏览器写json。

3. Object转JSON字符的方式，可以通过jar包，我用的是alibaba的fastJacson包。

   ```java
   @Override
   public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
           RequestMappingHandler rmHandler = (RequestMappingHandler)handler;
           Model model = new Model();
           Object[] args = ArgumentResolverUtil.resloveRequsetParam(request,rmHandler.getMethod(),model);
           Object obj = rmHandler.getMethod().invoke(rmHandler.getBean(),args);
           if(obj==null)
               return null;
           //@ResponseBody
           Annotation annotation = rmHandler.getMethod().getAnnotation(ResponseBody.class);
           if(annotation!=null){
               response.getWriter().write(JSONObject.toJSONString(obj));
               return null;
           }
           if(obj instanceof ModelAndView)
               return (ModelAndView)obj;
           ModelAndView mv = new ModelAndView();
           if(obj instanceof String){
               mv.setModel(model);
               mv.setView((String) obj);
           }
   
           return mv;
   }
   ```

   ```xml
   <dependency>
       <groupId>com.alibaba</groupId>
       <artifactId>fastjson</artifactId>
       <version>1.2.58</version>
   </dependency>
   ```

   ## 小结

   + git tag
     * step-1.1-solve-SpringInit Spring初始化
     * step-1.2-solve-SpringMVCInit MVC初始化
     * step-2.1-solve-requestMapping RequestMapping组件的实现
     * step-2.2-solve-interceptor 拦截器的实现
     * step-3.0-solve-step2Problems 解决第二步的少许问题
     * step-3.1-solveHandlerAdapter HandlerAdapter组件的实现
     * step-4.1-solveViewResolverAndView 视图及视图解析器组件的时间
     * step-4.2-solveResponseBody ResponseBody注解的实现