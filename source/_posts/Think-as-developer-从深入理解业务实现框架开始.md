---
title: 'Think as developer,从深入理解业务实现框架开始'
date: 2016-08-12 11:30:47
tags: 
- 测试思维
- java
categories: 测试
---
### 引言

这篇文章主要介绍笔者从经历过的项目中，看到自己或者项目组QA在产品迭代中的软肋。认为可以通过另外一种思路来改善这些弱点。希望能够有助于QA最大化的做好质量保障工作。


### 1，产品迭代中QA自身的Bug（软肋）是什么？
#### 1.1 需求偏于口头传述
产品的快速迭代，可以迅速满足用户需求，然而却也有一些后遗症，比如部分需求描述偏向于口头传述，文档后于实现，会给QA工作造成比较大的困难。所以在坚守一些固有可靠的测试流程之外，我们需要想一些新的办法。
#### 1.2 需求和实现衔接不能达到无缝连接
测试的起点是明确测试的需求，然而有些很具体的问题，需要帮助开发定位问题的时候，需求只定义了一些比较粗的业务目标，然而具体实现由开发掌握。这里的衔接过程，QA是袖手旁观呢，还是参与其中。笔者认为后者可能可以更深入的接近实现，达到最大化的质量保障。 

是时候开拓一种新思路了。**Think as developer**，从阅读并深入理解业务实现框架开始吧。即使代码并不是出自QA手，用到的开源或者内部框架并不熟悉，那么也要尝试开始跳出QA的舒适区，开始更贴合的去理解业务逻辑及代码实现细节。这个过程，QA要重新定位自己，重新定位质量保障的核心竞争力。

###  2，在项目中的具体实践

有了**Think as developer**这样的思路，那么如何具体到 **Do as developer**？笔者在下面2个项目中进行了实践：

- 某运维监控系统*Web*、*API*及*Task*模块源码阅读
- 某智能语音项目*Java*层源码阅读

> 鉴于目前接触到的大多PC端的项目，大多是*war*包形式走的*Tomcat*。所以这篇文章主要是面向这类产品的一些阅读方法和技巧总结进行的一些实践活动。

也提炼出来了下面3个具体的阅读技巧，可以更快速有效的作为理解业务实现框架的切入点：
- [**一个Web项目的容器启动的入口是什么？**](#jump1)
- [**深入Spring MVC + Mybatis的一些成熟的工程架构如何配置？**](#jump2)
- [**Spring 和SpringMVC是两件事**](#jump3)


下面的篇幅，来具体谈谈这3个阅读技巧。

####  <span id="jump1">2.1，一个Web项目的容器启动的入口是什么？</span>
记得最早的时候开始研究*Java Web*的时候，记得看到一个人写的一句话：
“初学 *Java Web* 开发，请远离各种框架，从 *Servlet* 开发” 
> [初学 Java Web 开发，请远离各种框架，从 Servlet 开发](http://www.oschina.net/question/12_52027)

*Java Web*开发离不开*Servlet*，*Servlet*的生命周期是有Tomcat/Jetty这样的Web容器接管，那么`web.xml`就是所有开始的入口。这里会配置*Servlet*，*Filter*这样的组件。

**Servlet**：通过*doGet doPost*方法处理请求，这个方法里有传统的两个入参：*HttpServletRequest，HttpServletResponse*来分别处理请求和响应。

**Filter**：在请求被容器发到servlet之前，会先经过配置的*filter*。所以一般情况下，*filter*都是做一些白名单验证，特定的*uri*要通过*openid*，*doFilter*方法在做。
这个时候`web.xml`里应该会有很多`<servlet>`和`<filter>`标签，杂乱无章

加入的*SpringMVC*框架后，`web.xml`就变得简化无比（只是web.xml），需要关注的有下面这些：

```java
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
        classpath:spring-context-web.xml
    </param-value>
    </context-param>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
```
以及*Servlet*的配置：

```java
    <servlet>
        <servlet-name>sentry</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-mvc-config.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
```

又出来了两个配置文件：*spring-context-web.xml* & *spring-mvc-config.xml*
一个是通用上下文，一个是初始化*MVC*上下文。如下图
 ![Alt text](./1.png)


那么各有什么用处：
- ***ContextLoaderListener***初始化的上下文加载的*Bean*是对于整个应用程序共享的，不管是使用什么表现层技术，一般如*DAO*层、*Service*层Bean；
- ***DispatcherServlet***初始化的上下文加载的*Bean*是只对*Spring Web MVC*有效的*Bean*，如*Controller*、*HandlerMapping*、*HandlerAdapter*等等，该初始化上下文应该只加载*Web*相关组件。

这里就可以大致知道，之前所有`<servlet>`需要做的事情，都被*Spring*的*Dispatcher servlet*统一接管，可以理解为一个虚拟的路由器，将请求转发给所有的`@Controller`。

这里碰见过一个事情：我有个外部的服务需要初始化，初始化如下：

```java
    <bean id="qaService" class="com.xxx.utils.qa.qaServiceImpl">
        <constructor-arg index="0" value="${baseURL}" />
        <constructor-arg index="1" value="${token}" />
    </bean>
```
我用它的地方是在一个*Controller*里面，然而放在*spring-mvc-config.xml*就编译失败，说找不到这个*Bean*。放在*spring-context-web.xml*就可以。


#### <span id="jump2">2.2，深入Spring MVC + Mybatis的一些成熟的工程架构如何配置？</span>
你的项目目录应该是这样的：

- **/main/**：
*controller*层，*service*层及*DAO*层，以及*filter*：

![Alt text](./2.png)


- **/Resources**：
使用*MyBatis*的话，这些*mapper*文件放在这里，并使用和*DAO*层一样的包名。
然后根据不同的开发，测试，线上环境放入不同的配置文件。
![Alt text](./3.png)


至于配置文件如何读取，一方面*Maven*编译打包的时候*resource*目录下的文件都会拷贝出来。另外一方面，区分环境变量，在`pom.xml`的`<profile>`配置即可，然后通过*mvn*的 -P参数来区分，如图：

```java
        <profile>
            <id>dev</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <build>
                <resources>
                    <resource>
                        <directory>src/main/resources/dev</directory>
                    </resource>
                    <resource>
                        <directory>src/main/resources/common</directory>
                    </resource>
                </resources>
            </build>
        </profile>
        <profile>
            <id>test</id>
            /* 同上 */
            <directory>src/main/resources/test</directory>
        </profile>
        <profile>
            <id>online</id>
            ...
            <directory>src/main/resources/online</directory>
            ...
            /* 同上 */
        </profile>
```
- **请求如何到达Controller**

这里有个关键注释：`<mvc:annotation-driven/>`
因为之前肯定是通过`<context:component-scan/>`扫描过所有的*Controller*，但是他们只是*Bean*被构造，需要通过`<mvc:annotation-driven/>`标签告诉*SpringMVC*，请求的处理者。
> 出处：[spring-mvc-difference-between-contextcomponent-scan-and-annotation-driven](http://stackoverflow.com/questions/13661985/spring-mvc-difference-between-contextcomponent-scan-and-annotation-driven)

- **请求返回**：*FTL*返回及*JSON*返回

这里要在*spring_mvc_config.xml*中配置一个*bean*：***ContentNegotiatingViewResolver***，有两个属性：

```java
<property name="defaultContentType" value="application/json" />
.....
<property name="viewResolvers">
.....
<property name="defaultViews">
```

这两个*resolver*的好处在于：*Controller*的函数处理，返回*String*就默认是*FTL*的路径（也就是前端的路径），返回*void*，就是*json*。不需要`@RequestBody`注解。

- ***Controller* 怎么看，怎么写**

了解一些关键注释的意思，比如`@RequestMapping`， `@RequestParam`,  `@RequestHeader ` ，`@PathVariable`， 至于`ResponseBody`，经过上面的讲解，应该就不需用了。别的话，多看代码，多实践，不缺这样的资料。

- **前端的配置**

前端这里有两个关键配置：

```java
<mvc:resources mapping="/views/**" location="/views/"/>`
<bean
            class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
        <property name="templateLoaderPath" value="/views"/>`
```
因为*Servlet*被*Spring*整体接管之后，所有的请求都被接管。那么静态文件呢？他们又没有*Controller*的处理，肯定会*404 not found*。
- 第一个注释就是解决了这个问题，配置了静态文件的本地路径`/views/`，这里根目录是*webapp*， 那么所有的*CSS，JS，PNG*, 都将在来这里找。
- 第二个注释其实就是配置*FreeMarker*的模版路径，一般工程也都放在*webapp*的`/views/`下。

#### <span id="jump3">2.3，Spring 和SpringMVC是两件事</span>

这里碰见过一个事情：我有个外部的服务需要初始化，初始化如下：


```java
    <bean id="qaService" class="com.xxx.utils.qa.qaServiceImpl">
        <constructor-arg index="0" value="${baseURL}" />
        <constructor-arg index="1" value="${token}" />
    </bean>
```

我用它的地方是在一个*Controller*里面，然而放在*spring-mvc-config.xml*就编译失败，说找不到这个*Bean*。具体为啥，可以参考上一节。因为这些*Bean*并不是有*SpringMVC*通过`@Service`标签来统一注入管理。那么它的初始化过程应该要放入*spring-context-web.xml*，在*SpringMVC*介入之间就需要实例化。否则当然只是一个接口。 

所以到这里为止，相信已经对此种类型的项目有了一个基本的阅读技巧。下一章，来说一下笔者在这些基础之上，实践得到的一些感悟。

#### <span id="jump">3，实践所得</span>

- **能够迅速上手项目，并快速理解项目基本逻辑及架构。**
在我新接手的一个某智能语音机器人项目中，在没有任何需求设计接口文档的情况下，要开展测试工作，只能先从源码开始，之前的经验帮助了我，理清楚了所有接口的处理逻辑（毕竟是半路接手项目，只能先解决问题为先）。基本上一两天功夫就可以把源码大体逻辑以及框架读懂（当然，大型的项目要花更长的时间）。当时的几个思维导图之一如下：
![Alt text](./4.png)

然后就开始欢快的写接口测试用例。

- **QA不仅仅要整体业务上有宏观把控。出现微观上bug，也能做出*Root Cause*的前瞻分析，能够迅速定位问题。**
- 略深入理解*Spring*及*SpringMVC*。
- 找到快速融入研发团队的切入点。其实似乎是没法量化的，然而个人在团队中起到的化学反应相信都能感受到。

####  结语

笔者认为在项目质量保障过程中，提升QA战斗力，需要开拓新思路。提出**Think as developer**，以及**Do as developer**在某一类项目中的一些具体技巧。为读者提供另外一种思维方式。

所以白盒与黑盒不是测试手段，而是测试思维。过度关注开发细节的白盒测试没有意义，从需求出发更加的符合实际中的测试。
> 引自 [链接](http://mp.weixin.qq.com/s?__biz=MzI2MjEwMDE2OQ==&mid=2648586125&idx=1&sn=e5416afcc83a23b485d6d3b75da41b7e&scene=1&srcid=0521DV7YsyaaWND9LrEuHeYF#rd)

**业务理解第一，业务理解第一，业务理解第一。**


