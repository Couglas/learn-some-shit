# Servlet

Servlet是web开发的原点，几乎所有的Java Web框架都是基于Servlet的封装。比如Spring本身就是一个Servlet。Tomcat和Jetty这样的容器，负责运行和加载Servlet

os->jvm->tomcat\jetty->spring->web应用

早期的Web应用用于浏览静态页面，HTTP服务器（Apache、Nginx）返回静态HTML给浏览器，浏览器解析HTML给用户。后来，需求增长为需要交互，需要扩展机制让HTTP服务器调用服务端程序，因此Sun公司推出了Servlet。Servlet简单可以理解为运行在服务端的Java小程序，但它没有main方法，不能独立运行，必须部署到Servlet容器中，由容器实例化并调用Servlet

Tomcat、Jetty=HTTP服务器+Servlet容器=Web容器

Tomcat、Jetty作为HTTP服务器，在请求中做了哪些事？

1. 接受连接
2. 解析请求数据
3. 处理请求
4. 发送响应

请求首先会打包成HTTP协议的格式：

1. 请求行：POST URL
2. 请求报头：Accept、Accept-Encoding、Accept-Language、Connection、Content-Length、Content-Type、Host、User-Agent
3. 请求正文：json 

响应HTTP协议的格式

1. 状态行：200
2. 响应报头：Connection、Content-Type、Date、Set-Cookie
3. 主体报文：json

HTTP无状态是指不同请求之间协议内容不想管，本次和上次请求内容无依赖，响应也是如此。而要保存用户信息则可使用Cookie和Session。

Cookie是本地明文存储的用户信息，每次请求可以携带

Session是服务端存储的信息，一般使用SessionId来识别用户

浏览器发送服务端一个HTTP格式的请求，HTTP服务器收到后调用服务端程序即Java类，一般来说不同的请求需要不同的Java类处理。如果通过大量ifelse处理，太过耦合，因此采用面向接口编程，业务类必须实现Servlet接口。但是还有一个问题，HTTP服务器不知道由哪个Servlet来处理，Servlet由谁实例化，且也不适合放到HTTP服务器中，这种情况下，引入了Servlet容器来处理此问题，Servlet容器会将请求转发到具体的Servlet，如果此Servlet还没创建，就加载并实例化，然后调用它的接口方法。流程就变为浏览器->HTTP服务器->Servlet容器->业务类

![image-20241231105620818](/Users/a123456/Library/Application Support/typora-user-images/image-20241231105620818.png)

Servlet接口和Servlet容器一整套规范叫做Servlet规范。Tomcat和Jetty都按照此规范实现了Servlet容器，同时也具备HTTP服务器的功能。

```java
public interface Servlet {
    
    void init(ServletConfig config) throws ServletException;
    // web.xml配置，这里可以获得
    ServletConfig getServletConfig();
    
    void service(ServletRequest req, ServletResponse res）throws ServletException, IOException;
    
    String getServletInfo();
    
    void destroy();
}
```

有接口一般有抽象类，用来实现接口和封装通用逻辑。GenericServlet就是其抽象类，可以扩展这个类实现Servlet。但这个与通信协议无关，大多Servlet在HTTP中处理，所以还有HttpServlet继承了GenericServlet，加入了HTTP特性，只需重写doGet和doPost方法。

## Servlet工作流程

![image-20241231110733938](/Users/a123456/Library/Application Support/typora-user-images/image-20241231110733938.png)

1. HTTP服务器封装一个ServletRquest
2. 调用Servlet容器的service方法
3. Servlet容器根据请求映射URL和Serlvet，找到响应的Servlet，如果没加载，反射创建它，调用init初始化，然后调用Servlet.service处理请求
4. 封装一个ServletResponse返回HTTP服务器

## Servlet如何注册到Servlet容器中？

以web应用的方式部署Servlet。根据Servlet规范，Web应用是以下的目录结构

```
webApp
	WEB-INF/web.xml 	-- 配置文件，配置Servlet等
	WEB-INF/lib/			-- 存放web应用所需的jar包
	WEB-INF/classes/	-- 存放应用类
	META-INF/ 				-- 存放工程的一些信息
```

Servlet规范定义了ServletContext接口对应一个Web应用。Web应用部署好之后，Servlet容器启动时会加载Web应用，并为其创建唯一的ServletContext对象。一个Web应用可能有多个Servlet，这些可以通过ServletContext共享数据

## Servlet的扩展机制

- filter：过滤器，允许对请求和响应做统一的定制化处理。比如限流、国际化
  - 其流程是：Web应用部署完成后，Servlet容器实例化Filter并链接成FilterChain，请求来临时，获取第一个Filter定调用doFilter，doFilter负责调用下一个Filter
- listener：监听器，配置在web.xml中
  - Web应用在Servlet容器运行时，产生各种事件，如请求到达、web应用启动停止等。Servlet容器提供了一些默认的监听器监听这些事件，当事件发生时，调用监听器的方法。Spring就实现了自己的监听器，监听ServletContext的启动事件，当Servlet容器启动时，创建并初始化全局的Spring容器

两者的区别：

- Filter是干预过程的，是过程的一部分，基于过程行为
- Listener是基于状态的，任何行为改变同一状态，触发同一事件

## Servlet容器、Spring容器、SpringMVC容器、Tomcat之间的关系

![image-20241231113017374](/Users/a123456/Library/Application Support/typora-user-images/image-20241231113017374.png)

Tomcat&Jetty在启动时给每个Web应用创建一个全局的上下文环境，这个上下文就是ServletContext，其为后面的Spring容器提供宿主环境。 Tomcat&Jetty在启动过程中触发容器初始化事件，Spring的ContextLoaderListener会监听到这个事件，它的contextInitialized方法会被调用，在这个方法中，Spring会初始化全局的Spring根容器，这个就是Spring的IoC容器，IoC容器初始化完毕后，Spring将其存储到ServletContext中，便于以后来获取。 Tomcat&Jetty在启动过程中还会扫描Servlet，一个Web应用中的Servlet可以有多个，以SpringMVC中的DispatcherServlet为例，这个Servlet实际上是一个标准的前端控制器，用以转发、匹配、处理每个Servlet请求。 Servlet一般会延迟加载，当第一个请求达到时，Tomcat&Jetty发现DispatcherServlet还没有被实例化，就调用DispatcherServlet的init方法，DispatcherServlet在初始化的时候会建立自己的容器，叫做SpringMVC 容器，用来持有Spring MVC相关的Bean。同时，Spring  MVC还会通过ServletContext拿到Spring根容器，并将Spring根容器设为SpringMVC容器的父容器，请注意，Spring MVC容器可以访问父容器中的Bean，但是父容器不能访问子容器的Bean，  也就是说Spring根容器不能访问SpringMVC容器里的Bean。说的通俗点就是，在Controller里可以访问Service对象，但是在Service里不可以访问Controller对象

# Tomcat

## 相关日志说明

- catalina.**.log：主要记录Tomcat启动过程的信息，可以看到启动的JVM参数及操作系统等日志
- catalina.log：Tomcat的标准输出，在tomcat的启动脚本里指定的，如果没有修改的话，stdout和stderr会重定向到这里
- localhost.**.log：记录web应用在初始化过程中遇到的未处理的异常，被tomcat捕获而输出
- localhost_access_log.**.txt：tomcat的请求日志，包括ip地址、请求路径、时间、协议、状态码等
- manager.**.log：自带的manager项目的日志

## 支持的IO模型

- NIO：非阻塞I/O，JavaNiO类库实现
- NIO.2：异步IO，JDK7最新的NIO.2类库实现
- APR：Apache可移植运行库，C/C++编写的本地库

## 支持的应用层协议

- HTTP/1.1：大部分web应用采用的

- AJP：用于和web服务器集成，如Apache

- HTTP/2：提升了web性能

  

## 总体架构

- 处理socket连接，负责网络字节流与Rquest、Response对象的转换
- 加载和管理Servlet，处理具体的Rquest请求

![image-20241231150954441](/Users/a123456/Library/Application Support/typora-user-images/image-20241231150954441.png)

两个重要组件用来处理以上两个核心功能

### 连接器Connector

连接器对Servlet容器屏蔽了协议及IO模型等的区别，在容器中获得的都是标准的ServletRequest对象

- 功能划分

  - 监听网络端口
  - 接口网络连接请求
  - 读取网络请求字节流
  - 根据应用层协议（HTTP、AJP）解析字节流，生成统一的Tomcat Request对象
  - 将Tomcat Requestt对象转成标准的ServletRequest
  - 调用Servlet容器，得到ServletResponse
  - 将ServletResponse转成Tomcat Response对象
  - 将Tomcat Response对象转成网络字节流
  - 将字节流写回浏览器

- 内聚以上功能得到

  - 网络通信：Endpoint

    实现TCP/I协议的，通信端点。是一个接口，抽象类是AbstractEndPoint，具体子类是NioEndPoint和Nio2EndPoint

    	1. Acceptor：监听Socket连接请求
    	1. SocketProcessor：处理接收到的Socket请求，实现Runnable接口，在run里调Processor处理，被提交到线程池Executor执行

    ![image-20241231152230295](/Users/a123456/Library/Application Support/typora-user-images/image-20241231152230295.png)

  - 应用层协议解析：Processor

    实现HTTP协议，接口来自EndPoint的Socket，读取字节流解析成Tomacat Request和Response对象，通过Adapter提交到容器处理

    ![image-20241231153115218](/Users/a123456/Library/Application Support/typora-user-images/image-20241231153115218.png)

  - Tomcat Request/Response和ServletRequest/ServletResponse转换：Adapter

    处理Tomcat自定义的请求和响应对象到标准的Servlet响应和对象，通过适配器CoyoteAdapter.service进行转换

![image-20241231152127619](/Users/a123456/Library/Application Support/typora-user-images/image-20241231152127619.png)

Tomcat 的整体架构包含了两个核心组件连接器和容器。连接器负责对外交流，容器负责内部处理。连接器用 ProtocolHandler 接口来封装通信协议和 I/O 模型的差异，ProtocolHandler 内部又分为 Endpoint 和 Processor 模块，Endpoint 负责底层 Socket 通信，Processor 负责应用层协议解析。连接器通过适配器 Adapter 调用容器。

### 容器Container

![image-20241231153624302](/Users/a123456/Library/Application Support/typora-user-images/image-20241231153624302.png)

Context 表示一个 Web 应用程序；Wrapper 表示一个 Servlet，一个 Web 应用程序中可能会有多个 Servlet；Host 代表的是一个虚拟主机，或者说一个站点，可以给 Tomcat 配置多个虚拟主机地址，而一个虚拟主机下可以部署多个 Web 应用程序；Engine 表示引擎，用来管理多个虚拟站点，一个 Service 最多只能有一个 Engine。Tomcat通过组合模式管理这些容器。

```java
public interface Container extends Lifecycle {
    public void setName(String name);
    public Container getParent();
    public void setParent(Container container);
    public void addChild(Container child);
    public void removeChild(Container child);
    public Container findChild(String name);
}
```

![image-20241231153841204](/Users/a123456/Library/Application Support/typora-user-images/image-20241231153841204.png)

定位servlet，Tomcat使用Mapper组件

![image-20241231154625499](/Users/a123456/Library/Application Support/typora-user-images/image-20241231154625499.png)

Tomcat 通过一层一层的父子容器找到某个 Servlet 来处理请求。需要注意的是，并不是说只有 Servlet 才会去处理请求，实际上这个查找路径上的父子容器都会对请求做一些处理，其调用过程使用 Pipeline-Valve 责任链模式实现

![image-20241231155158310](/Users/a123456/Library/Application Support/typora-user-images/image-20241231155158310.png)

Valve是一个处理节点，invoke是用来处理请求的。Pipeline中维护了Valve链表。每一个容器中都有一个Pipeline对象，触发其第一个Valve，容器中的Pipeline的Valve都会被调用到

```java
public interface Valve {
  public Valve getNext();
  public void setNext(Valve valve);
  public void invoke(Request request, Response response)
}

public interface Pipeline extends Contained {
  public void addValve(Valve valve);
  public Valve getBasic();
  public void setBasic(Valve valve);
  public Valve getFirst();
}
```

Wrapper容器的最后一个Valve回创建一个Filter链，并调用doFilter方法，最后一个fileter负责调用Servlet.service方法

![image-20241231164152093](/Users/a123456/Library/Application Support/typora-user-images/image-20241231164152093.png)

valve和filter有什么区别？

1. valve是Tomcat的私有机制，与Tomcat的基础架构、API紧耦合。ServletAPI是公有标准，所有Web容器都支持Filter机制
2. valve工作在web容器级别，拦截所有应用请求；而filter工作在应用级别，只能拦截某个web应用的所有请求



## 如何实现一键式启停

- Lifecycle接口：init、start、stop、destory。所有组件都实现这个接口，父组件的init调用子组件的init，父组件的start调用子组件的start，即组合模式，这样只需调用顶层组件，Server的init和start，Tomcat就启动起来了

- Lifecycle事件：组件的init和start调用由父组件的状态变化触发的，上层组件的init会触发子组件的init。因此把组件的声明周期定义成一个个状态，状态的转变看做事件，事件可以通过监听器处理，也方便添加和删除，应用到观察者模式。在LifeCycle接口里面添加addLifecycleListener和removeLifecycleListener方法。其中LifeCycleState包含：NEW、INITIALIZING、INITIALIZED、STARTING_PREP、STARTING、STARTED、STOPPING_PREP、STOPPING、STOPPED、DESTORING、DESTORYED、FAILED

- LifecycleBase抽象类：复用某些逻辑，如生命状态的转变与维护、生命事件的触发以及监听器添加和删除等，而子类就负责自己的初始化、启动和停止，由子类去继承这个类

  ```java
  public abstract class LifecycleBase implements Lifecycle {
    abstract void initInternal();
    abstract void startInternal();
    abstract void stopInternal();
    abstract void destoryInternal();
    
    @Override
    public final synchronized void init() throws LifecycleException {
        //1. 状态检查
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
        }
  
        try {
            //2.触发INITIALIZING事件的监听器
            setStateInternal(LifecycleState.INITIALIZING, null, false);
  
            //3.调用具体子类的初始化方法
            initInternal();
  
            //4. 触发INITIALIZED事件的监听器
            setStateInternal(LifecycleState.INITIALIZED, null, false);
        } catch (Throwable t) {
          ...
        }
    }
  }
  
  ```

  监听器是什么时候注册进来的？谁注册的？

  1. Tomcat自定义了一些监听器，这些是父组件在创建子组件的过程中注册到子组件的，如MeroryLeakTrackingListener，检测Context容器中的内存泄露，它是在Host容器创建Context容器时注册到Context的
  2. server.xml中定义自己的监听器，Tomcat在启动时会解析server.xml，创建并注册到容器组件

## 高层组件都负责什么?

![image-20241231175304630](/Users/a123456/Library/Application Support/typora-user-images/image-20241231175304630.png)

- Catalina：创建Server，解析server.xml，将其中的组件一一创建出来，然后调用Server.init和start。除此以外，还向JVM注册了一个关闭钩子，一个线程调用stop，释放和清理所有资源

- Server：具体实现是StandardServer。继承了LifecycleBase，子组件是Servce，启动时调用Service的启动，停止调用Service的停止。内部维护了若干Service组件，用数组保存，添加的过程中动态扩展。最后，Catalina调用了Server.await，在await方法里创建一个监听8005的Sockect，在死循环里接收Socket上的链接请求，如果有新连接就建立连接，然后从Socket中读取数据，如果是SHUTDOWN，就退出循环，执行stop

- Service：具体实现是StandardService。继承了LifecycleBase，其中还有Server、Connector、Engine、Mappper。MapperListener用来监听容器的变化，并将信息更新到Mapper中，使Tomcat支持热部署。维护其他组件的生命周期，基于其他组件的依赖关系：先启动Engine，再启动Mapper监听器，最后启动Connector。因为内层组件启动好了才能对外提供服务，然后才启动外层的连接器，而Mapper依赖容器组件，容器组件启动好了才能监听他们的变化，因此Mapper和MapperListener在容器之后才启动。Engine和Connector会启动他们的子组件。组件停止跟启动顺序是相反的

- Engine：具体实现是StandardEngine。本质是一个容器，继承了ContanierBase，实现Engine接口。包含子容器Host的数组，在ContainerBase中实现了，使用HashMap存储，使用专门线程池启动子容器，实现了子容器增删改查，启动停止默认实现，Engine直接使用这些。它自己主要是处理请求，即转发给Host子容器，通过Valve实现的。请求到达Engine容器之前，Mapper组件已经对请求进行了路由处理，定位了响应的容器，并保存到了请求对象中

  

## 如何优化Tomact启动速度

1. 清理不必要的web应用。删除webapps目录下不需要的工程
2. 清理xml配置文件
3. 清理jar文件。web应用中的lib目录不应该出现servlet api或tomcat自身的jar，这些由tomcat负责提供。使用maven的话依赖指定为provided
4. 清理其他文件。即使清理日志文件
5. 禁止Tomcat TLD扫描。如果不需要支持JSP的话，不用扫描jar里的TLD文件，加载里面的标签库
   1. conf/context.xml，加上jarScanner和JarScanFilter标签，defaultTldScan=false
   2. 如果必须使用jsp，则conf/properties，添加：tomcat.util.scan.StandardJarScanFilter.jarsToSkip=xxx.jar
6. 关闭WebSocket支持。Tomcat会扫描WebSocket注解的API实现，如@ServerEndPoint注解的类。注解扫描一般比较慢
   1. conf/context.xml，context标签加containerSciFilter=org.apache.tomcat.websocket.server.WsSci
   2. 如果不需要，直接删除lib目录下的websocket-api.jar和tomcat-websocket.jar
7. 关闭JSP支持
   1. conf/context.xml，context标签加containerSciFilter=org.apache.jasper.servlet.JasperInitializer | ...
8. 禁止Servlet注解扫描。不使用Servlet这个注解。
   1. web.xml，设置web-app元素的属性metadata-complete=true，告诉tomcat web.xml配置的Servlet是完整的，无需找类中的Servlet定义
9. 配置Web-Fragment扫描。这是一个部署描述文件，可以完成web.xml的配置功能。而这个web-fragment.xml文件必须存放在 JAR 文件的META-INF目录下，而 JAR 包通常放在WEB-INF/lib目录下，因此 Tomcat 需要对 JAR 文件进行扫描才能支持这个功能
   1. web.xml的absolute-ordering元素指定哪些jar需要扫描
10. 随机数熵源优化。Tomcat 7 以上的版本依赖 Java 的 SecureRandom 类来生成随机数，比如 Session ID。而 JVM 默认使用阻塞式熵源（/dev/random）， 在某些情况下就会导致 Tomcat 启动变慢
    1. 设置JVM参数，-Djava.security.egd=file:/dev/./urandom。使用非阻塞熵源安全较低，可以考虑使用硬件方式
11. 并行启动多个Web应用。默认是一个一个启动
    1. server.xml中Host元素startStopThreads属性设置，表示多少个线程来启动web应用
12. 其他
    1. 调大vm xms xmx避免反复扩容堆内存 换上固态硬盘可以提速xml文件读取 server.xml去掉监听 去掉不要的ajp 去掉多余的连接器 线程池的核心线程设置延迟初始化 去掉access log，因为nginx里已有access log 减少项目里多余的jar 精确设置mvc注解的包扫描范围 xml spring bean设置延迟初始化 数据库连接池初始化数量减少

## NioEndPoint组件：实现非阻塞I/O

I/O是计算机内存与外部设备之间拷贝数据的过程。cpu访问内存速度远高于外部设备，cpu先把外部设备的数据读到内存里再操作，前一步操作耗时，cpu空闲，此时如何处理是I/O模型要解决的问题

UNIX支持的几种I/O：

- 同步阻塞：用户线程发起read后阻塞，让出cpu，等待内核准备数据、拷贝数据，直到返回唤醒用户线程
- 同步非阻塞：用户线程发起read后，不断read，失败立即返回，成功阻塞数据拷贝，直到返回唤醒用户线程
- 多路复用：和同步非阻塞一样，不过通过select不断查询，监听多个channel
- 信号驱动：不同于同步非阻塞不断read，它是发起read后注册信号处理函数，算回调，数据到内核后，内核触发该函数，再调一次read。是个半异步的过程。
- 异步：调用read时注册一个回调函数，不阻塞，完成后回调函数进行处理

Java实现了除信号驱动之外的I/O模型。NioEndPoint则实现了多路复用模型。

多路复用无非就是两步：1. 创建selector，在其上注册感兴趣的事件，然后调select方法，等待事件发生 2. 事件发生，比如可读了，就创建线程从channel读取

NioEndPoint包含LimitLatch、Acceptor、Poller、SocketProcessor、Executor5个组件

![image-20250110104723810](/Users/a123456/Library/Application Support/typora-user-images/image-20250110104723810.png)

- LimitLatch：连接控制器，负责控制最大连接数，NIO模式默认10000，达到阈值，新请求拒绝。通过内部类Sync（实现了AQS并重写了它的tryAcquireShared和tryReleaseShared）来控制连接数，volatile修饰的limit，控制线程什么时候挂起，什么时候唤醒
- Acceptor：跑在一个单线程里，死循环调用accept接收新连接，一旦有新的到来，accept方法返回一个SocketChannel对象，封装成PollerEvent中，压入Poller的队列里，交给Poller处理。ServerSocketChannel在多个Acceptor线程之间共享，是EndPoint的属性，由EndPoint初始化和端口绑定
- Poller：本质是一个Selector，跑在单线程里。其内部维护一个Channel数组（Queue的形式），在死循环里不断检测Channel的数据就绪状态，一旦有Channel可读，生成一个SocketProcessor任务对象扔给Executor执行。另一个任务是循环遍历管理的SocketChannel是否超时，如果超时就关闭该channel
- Executor：Tomcat定制版的线程池，负责运行SocketProcessor任务类，SocketProcessor的run方法调用Http11Processor（应用层协议的封装）读取解析请求数据，调用容器获取响应，把响应通过Channel写出

高并发思路：高并发就是能快速处理大量请求，需要合理设计线程模型让cpu忙起来，尽量不要让线程阻塞，一旦阻塞，cpu就闲了。有多少任务，就用相应规模的线程数处理。NioEndPoint要完成接收连接、检测I/O事件、处理请求，因此它将三件事分开，用不同规模的线程数处理。且都可以进行配置。

## Nio2EndPoint组件：实现异步I/O

![image-20250110165412215](/Users/a123456/Library/Application Support/typora-user-images/image-20250110165412215.png)

- LimitLatch：连接控制器
- Nio2Acceptor：扩展了Acceptor，用异步I/O方式接收连接，通过回调函数来完成，跑在一个单线程里，也是一个线程组。Nio2Acceptor接收新的连接后，得到一个AsynchronousSocketChannel，封装成Nio2SocketWrapper，并创建SocketProcessor交给线程池处理，SocketProcessor持有Nio2SocketWrapper对象。
- Executor：执行SocketProcessor时，SocketProcessor的run方法会调用Http11Processor处理请求，通过Nio2SocketWrapper读取和解析请求数据，请求经过容器处理后，再把响应通过Nio2SocketWrapper写出
- Nio2SocketWrapper：用于封装Channel，提供接口给Http11Processor读写数据。Http11Processor不能阻塞等待数据，按异步I/O思路，Http11Processor调用Nio2SocketWrapper的read方法时注册回调类，read调用立即返回，此时Http11Processor还没读到数据，如何解决？通过两次read调用来完成
  - 第一次：连接刚建立好后，Acceptor创建SocketProcessor任务交给线程池处理，Http11Processor处理请求过程中，调用Nio2SocketWrapper的read发出第一次请求，同时注册readCompletationHandler回调类。此时数据没读到，Http11Processor将当前的Nio2SocketWrapper标记为数据不完整。然后Socketprocessor线程回收，Http11Processor不阻塞等待数据。
  - 第二次：当数据到达后，内核把数据拷贝到Http11Processor指定的Buffer里，同时回调类readCompletationHandler被调用，此回调处理创建一个新的SocketProcessor任务处理连接，但是持有原来的Nio2SocketWrapper，这次Http11Processor可以通过Nio2SocketWrapper读取数据，数据此时在应用层的Buffer中

与NioEndPoint不同的是，没有Poller组件（没有Selector），因为在异步I/O模式下，Selector工作是内核来做

## AprEndPoint组件

APR是Apache可移植运行时库，用C语言实现的，目的是向上层应用提供一个跨平台的操作系统接口库。Tomcat可以用它出来包括文件、网络I/O，从而提升性能。与NioEndPoint不同的是，NioEndPoint是通过调用Java的NIO API实现非阻塞I/O，AprEndPoint通过JNI（Java Native Interface，JDK提供的变成接口，允许Java调用其他语言编写的程序或库）调用APR本地库实现非阻塞I/O

频繁与操作系统进行交互，如Socket网络通信场景，Java比C的性能有差距，尤其是Web应用使用了TLS来加密传输，握手时有多次网络交互

![image-20250110173432328](/Users/a123456/Library/Application Support/typora-user-images/image-20250110173432328.png)

与NioEndPoint不同的只有Acceptor和Poller

- Acceptor：监听连接，接收并建立连接。其本质是调用了四个操作系统API：Socket、Bind、Listen和Accept。通过JNI调用。封装一个Java类，用native关键字修饰的方法

- Poller：不是通过Selector查询Socket状态，而是通过JNI调用APR中的poll方法，底层是调用了操作系统的epoll API实现的

  有个deferAccpet参数，对应TCP协议中的TCP_DEFFER_ACCEPT，设置后当客户端有新连接时，服务端先不建立连接，等数据来才建立。这样服务端不用Selector反复查询数据是否就绪

APR性能的秘密

1. C程序库

2. 使用DirectByteBuffer接收数据

   操作系统创建进程执行Java可执行程序，每个进程都有自己的虚拟地址空间，JVM用到的内存就是从进程的虚拟地址空间分配的，但只是进程空间的一部分，除此之外进程还有代码段、数据段、内存映射区、内核空间等。从JVM角度来看，JVM内存之外的部分叫本地内存

   ![image-20250110174210166](/Users/a123456/Library/Application Support/typora-user-images/image-20250110174210166.png)

   Tomcat 的 Endpoint 组件在接收网络数据时需要预先分配好一块 Buffer，所谓的 Buffer 就是字节数组byte[]，Java 通过 JNI 调用把这块 Buffer 的地址传给 C 代码，C 代码通过操作系统 API 读取 Socket 并把数据填充到这块 Buffer。Java NIO API 提供了两种 Buffer 来接收数据：HeapByteBuffer 和 DirectByteBuffer，两者的区别如下：

   - HeapByteBuffer对象本身在JVM分配，持有的字节数组也在JVM分配。使用它接收，数据会先从内核拷贝到临时的本地内存，再从临时的本地内存拷贝到JVM堆上。因为拷贝的过程中可能GC，GC后Buffer地址失效。而加上这一步，保证不会GC
   - DirectByteBuffer对象本身在JVM堆上，但持有的字节数不是JVM分配的，而是从本地内存分配的。其包含一个long类型的字段address，记录本地内存的地址。可以直接拷贝到本地内存且JVM可以直接读取该内存，速度比上一个快很多。不过本地内存地址不好管理，泄露难以定位

3. sendfile

   网络通信的场景中，静态文件收发：从磁盘读取文件到内存，将内存通过socket发送出去

   在传统的方式下，会存在多次拷贝，如下所示

   ![image-20250110175433059](/Users/a123456/Library/Application Support/typora-user-images/image-20250110175433059.png)

   经过6次内存拷贝，并且read和write等系统调用会切换用户态和内核态，耗费大量的cpu和内存资源

   Tomcat的AprEndPoint通过操作系统层面的sendfile解决了上面的问题

   ```java
   sendfile(socket, file, len);
   ```

   参数socket和文件句柄，将文件从磁盘写入socket只有两步：

   1.  文件读取到内核缓冲区
   2. 数据不从内核缓冲区复制到Socket关联的缓冲区，只将数据位置和长度描述符添加到Socket缓冲区中，然后直接从内核缓冲区传递给网卡

   ![image-20250110180024119](/Users/a123456/Library/Application Support/typora-user-images/image-20250110180024119.png)

## Executor组件：Tomcat如何扩展Java线程池

```java
//定制版的任务队列
taskqueue = new TaskQueue(maxQueueSize);

//定制版的线程工厂
TaskThreadFactory tf = new TaskThreadFactory(namePrefix,daemon,getThreadPriority());

//定制版的线程池
executor = new ThreadPoolExecutor(getMinSpareThreads(), getMaxThreads(), maxIdleTime, TimeUnit.MILLISECONDS,taskqueue, tf);
```

定制版的ThreadPoolExecutor，有如下关键点

- 有自己的定制版任务队列和线程工厂，并且可以限制任务队列的长度，最大长度是maxQueueSize

- 限制了核心线程数和最大线程数

- 重写execute方法定制了自己的任务处理流程，

  - 前corePoolSize个任务时，来一个任务就创建一个线程
  - 再来任务的话，就把任务添加到任务队列中让所有线程抢，如果队列满了就创建临时线程
  - 如果总线程数达到maxPoolSize，则继续尝试把任务添加到任务队列中
  - 如果缓冲队列也满了，插入失败，执行拒绝策略

  前两步一样，后面两步有区别

  ```java
  public class ThreadPoolExecutor extends java.util.concurrent.ThreadPoolExecutor {
    
    ...
    
    public void execute(Runnable command, long timeout, TimeUnit unit) {
        submittedCount.incrementAndGet();
        try {
            //调用Java原生线程池的execute去执行任务
            super.execute(command);
        } catch (RejectedExecutionException rx) {
           //如果总线程数达到maximumPoolSize，Java原生线程池执行拒绝策略
            if (super.getQueue() instanceof TaskQueue) {
                final TaskQueue queue = (TaskQueue)super.getQueue();
                try {
                    //继续尝试把任务放到任务队列中去
                    if (!queue.force(command, timeout, unit)) {
                        submittedCount.decrementAndGet();
                        //如果缓冲队列也满了，插入失败，执行拒绝策略。
                        throw new RejectedExecutionException("...");
                    }
                } 
            }
        }
  }
  ```

  Tomcat线程池用submittedCount来维护已经提交到线程池，但是还没执行完的任务个数。Tomcat的任务队列TaskQueue扩展了LinkedBlockingQueue（默认无长度限制，除非给capacity），重写了offer方法，在合适的时机返回false，代表任务添加失败。TaskQueue构造函数中有整型的capacity，传给了父类的LinkedBlockingQueue构造函数。capacity值来自Tomcat的maxQueueSize，这个值默认是无限大

  ```java
  public class TaskQueue extends LinkedBlockingQueue<Runnable> {
  
    ...
     @Override
    //线程池调用任务队列的方法时，当前线程数肯定已经大于核心线程数了
    public boolean offer(Runnable o) {
  
        //如果线程数已经到了最大值，不能创建新线程了，只能把任务添加到任务队列。
        if (parent.getPoolSize() == parent.getMaximumPoolSize()) 
            return super.offer(o);
            
        //执行到这里，表明当前线程数大于核心线程数，并且小于最大线程数。
        //表明是可以创建新线程的，那到底要不要创建呢？分两种情况：
        
        //1. 如果已提交的任务数小于当前线程数，表示还有空闲线程，无需创建新线程
        if (parent.getSubmittedCount()<=(parent.getPoolSize())) 
            return super.offer(o);
            
        //2. 如果已提交的任务数大于当前线程数，线程不够用了，返回false去创建新线程
        if (parent.getPoolSize()<parent.getMaximumPoolSize()) 
            return false;
            
        //默认情况下总是把任务添加到任务队列
        return super.offer(o);
    }
    
  }
  ```

  只有当前线程数大于核心线程数、小于最大线程数，并且已经提交的任务个数大于当前线程数时，说明线程不够用，但是线程数没达到极限，所以会创建线程。

  综上，维护submittedCount的目的是当队列无限大的时候让线程池有机会创建新的线程

## Tomcat如何支持WebSocket

- WebSocket

  网络中通信的双向链路的一段成为一个Socket，一个Sokect对应一个IP地址和端口号，应用程序通过Socket像网络发出请求或响应，它是对TCP/IP协议抽象出来的API。而WebSocket是一个应用层协议，通过HTTP协议进行一次握手，之后数据从Socket传输。浏览器发给服务端的请求和服务端的响应会带上跟WebScoket有关的请求头，如Connection:Upgrade 和Upgrade:websocket。建立好连接之后，数据传输以frame形式传输，将消息分为几个frame按顺序输出，一是可以不用考虑数据大小的问题，二是可以边生成边传输提升效率

Java WebSocket应用由一系列WebSocket Endpoint组成。Tomcat会给每个WebSocket连接创建一个EndPoint实例。有两种方式实现EndPoint：

1. 编程式：继承javax.websocket.Endpoint，实现onOpen、onClose、onError这些生命周期相关方法。Tomcat负责关联Endpoint生命周期并调用这些方法。当浏览器连接到一个EndPoint时，Tomcat给这个连接创建唯一的javax.websocket.Session（本质是对Socket的封装，Endpoint通过它与浏览器通信）。WebSocket连接握手成功时创建，连接关闭时销毁。当触发Endpoint各个生命周期事件时，Tomcat会将当前Session作为参数传给Endpoint的回调方法。通过在Session中添加MessageHandler接收消息，onMessage方法。
2. 注解式：实现业务类并添加@ServiceEndpoint(value = "url")

Tomcat在上面的过程中，主要做了两件事：EndPoint加载和WebSocket请求处理

- WebSocket加载：Tomcat通过SCI（ServletContainerInitializer，接收Web应用启动事件的接口）机制加载，通过监听Servlet容器的启动事件，在Web应用启动时做一些初始化工作，比如扫描和加载Endpoint类。SCI的使用很简单，通过实现ServletContainerInitializer接口并添加HandlsTypes注解，在注解中指定一系列类和接口集合。定义好SCI后，Tomcat在启动阶段扫描类，将HandlesTypes指定的类扫描出来，作为SCI的onStartUp方法的参数并调用。启动方法会构造一个WebSocketContainer实例（一个专门处理WebSOcket请求的Endpoint容器），Tomcat回吧扫描到的Endpoint子类和添加了@ServerEndpoint的类注册到容器中，并维护URL到Endpoint的映射，一次处理WebSocket请求

![image-20250113135606743](/Users/a123456/Library/Application Support/typora-user-images/image-20250113135606743.png)

这里的Endpoint和TOmcatProtocalHandler中的Endpoint不是一回事，后者是专门处理I/O通信

WebSocket通过HTTP协议进行握手，当WebSocket握手请求到来时，HttpProtocolHanlder首先接收到，处理时，Tomcat通过特殊Filter判断当前HTTP请求是否以诗歌WebSocket Upgrade请求（是否包含相应的HTTP头），是的话再响应里添加相应的头信息，并升级协议。具体来说就是用UpgradeProtocalHandler替换当前的HttpProtocolHanlder，将当前Socket的Processor换成UpgradeProcessor，同时Tomcat创建WebSocket Session实例和Endpoint实例并和当前WebSocket连接一一对应起来。

因此，Tomcat对WebSocket请求的处理没经过Servlet容器，通过UpgradeProcessor组件直接把请求发到ServerEndpoint实例。Tomcat的WebSocket视线不需要关注具体I/O模型的细节，从而实现了与具体I/O方式的解耦

## 对象池技术

用空间换时间。由于维护对象池本身也需要资源，所以不是所有场景都适合对象池。适用于对象数量很多且存在时间短，对象本身比较大，初始化成本高。而且对象池作为全局资源，高并发环境中多个线程可能同时需要获取对象池中的对象，会导致锁竞争而阻塞，因此也有线程同步的开销。

设计对象池

1. 需要尽量做到无锁化，如果内存足够大，可以考虑用ThreadLocal对象池，每个线程都有自己的对象池，互不干扰。
2. 池的大小有所限制，太小发挥不了作用，太大有空闲对象造成浪费。或许还要有自动扩容和自动缩容
3. 面对内存泄露的问题：池本质是一个Java集合类，集合类持有缓存对象的引用，集合类不GC，缓存对象也不会GC。大量的对象比较占用内存。必要时需要主动清理这些对象。如ThreadPoolExecutor有allowCoreThreadTimeOut和setKeepAliveTime，超时销毁线程。

使用对象池

1. 用完后，调用对象池的方法归还给对象池
2. 再次使用时需要重置，防止脏数据，脏对象的引用，导致意想不到的问题
3. 对象还给对象池后，不再操作此对象
4. 从池获取对象时，可能null、异常、阻塞等，需要额外的处理，确保健壮性

Tomcat使用SynchronizedStack实现对象池

```java
public class SynchronizedStack<T> {

    //内部维护一个对象数组,用数组实现栈的功能
    private Object[] stack;

    //这个方法用来归还对象，用synchronized进行线程同步
    public synchronized boolean push(T obj) {
        index++;
        if (index == size) {
            if (limit == -1 || size < limit) {
                expand();//对象不够用了，扩展对象数组
            } else {
                index--;
                return false;
            }
        }
        stack[index] = obj;
        return true;
    }
    
    //这个方法用来获取对象
    public synchronized T pop() {
        if (index == -1) {
            return null;
        }
        T result = (T) stack[index];
        stack[index--] = null;
        return result;
    }
    
    //扩展对象数组长度，以2倍大小扩展
    private void expand() {
      int newSize = size * 2;
      if (limit != -1 && newSize > limit) {
          newSize = limit;
      }
      //扩展策略是创建一个数组长度为原来两倍的新数组
      Object[] newStack = new Object[newSize];
      //将老数组对象引用复制到新数组
      System.arraycopy(stack, 0, newStack, 0, size);
      //将stack指向新数组，老数组可以被GC掉了
      stack = newStack;
      size = newSize;
   }
}
```

为什么不使用并发容器ConcurrentLinkedQueue？

1. 使用数组维护对象，减少结点维护内存开销
2. 只支持扩容不支持缩容，数组在使用时不会重新赋值，不会被GC。最低内存实现无界容器
3. Tomcat有最大请求数，不会无限膨胀

## Host容器：Tomcat如何实现热部署和热加载

如何实现热部署和热加载？

- 热加载的实现方式是Web容器启动一个后台线程，定期检测类文件的变化，变化就重新加载类，这个过程中不会清空Session，一般用在开发环境
- 热部署也类似，后台线程定时检测Web应用变化，但它重新加载整个Web应用，会清空Session，一般用在生产环境

Tomcat使用ScheduledThreadPoolExecutor开启后台线程

```java
// 开启后台线程
bgFuture = exec.scheduleWithFixedDelay(
              new ContainerBackgroundProcessor(),//要执行的Runnable
              backgroundProcessorDelay, //第一次执行延迟多久
              backgroundProcessorDelay, //之后每次执行间隔多久
              TimeUnit.SECONDS);        //时间单位

// ContainerBackgroundProcessor实现
protected class ContainerBackgroundProcessor implements Runnable {

    @Override
    public void run() {
        //请注意这里传入的参数是"宿主类"的实例
        processChildren(ContainerBase.this);
    }

    protected void processChildren(Container container) {
        try {
            //1. 调用当前容器的backgroundProcess方法。
            container.backgroundProcess();
            
            //2. 遍历所有的子容器，递归调用processChildren，
            //这样当前容器的子孙都会被处理            
            Container[] children = container.findChildren();
            for (int i = 0; i < children.length; i++) {
            //这里请你注意，容器基类有个变量叫做backgroundProcessorDelay，如果大于0，表明子容器有自己的后台线程，无需父容器来调用它的processChildren方法。
                if (children[i].getBackgroundProcessorDelay() <= 0) {
                    processChildren(children[i]);
                }
            }
        } catch (Throwable t) { ... }
```

ContainerBackgroundProcessor将ContainerBase的实例当成参数传给run。先调用当前容器的backgroundProcess，然后递归调用子孙的backgroundProcess，而backgroundProcess是Container容器的方法，有默认实现，因此只要实现了此接口的容器（Engine、Host、Context、Wrapper）都可以实现该方法完成周期性执行的任务，如果子类没实现则复用父类方法

```java
// 默认实现
public void backgroundProcess() {

    //1.执行容器中Cluster组件的周期性任务
    Cluster cluster = getClusterInternal();
    if (cluster != null) {
        cluster.backgroundProcess();
    }
    
    //2.执行容器中Realm组件的周期性任务
    Realm realm = getRealmInternal();
    if (realm != null) {
        realm.backgroundProcess();
   }
   
   //3.执行容器中Valve组件的周期性任务
    Valve current = pipeline.getFirst();
    while (current != null) {
       current.backgroundProcess();
       current = current.getNext();
    }
    
    //4. 触发容器的"周期事件"，Host容器的监听器HostConfig就靠它来调用
    fireLifecycleEvent(Lifecycle.PERIODIC_EVENT, null);
}
```

不仅每个容器可以有周期性任务，每个容器中的其他通用组件，如与集群管理有关的Cluster和跟安全管理的Realm组件都可以有自己的周期性任务，甚至Valve也可以有，都被ContainerBase统一处理。最后触发了容器的周期事件，这是一种扩展机制，容器存活一段时间后，如果想做点什么，就可以创建监听器监听这个事件，事件到了后执行相应的方法

Tomcat的热加载就是在Context容器中实现的

```java
public void backgroundProcess() {

    //WebappLoader周期性的检查WEB-INF/classes和WEB-INF/lib目录下的类文件
    Loader loader = getLoader();
    if (loader != null) {
        loader.backgroundProcess();        
    }
    
    //Session管理器周期性的检查是否有过期的Session
    Manager manager = getManager();
    if (manager != null) {
        manager.backgroundProcess();
    }
    
    //周期性的检查静态资源是否有变化
    WebResourceRoot resources = getResources();
    if (resources != null) {
        resources.backgroundProcess();
    }
    
    //调用父类ContainerBase的backgroundProcess方法
    super.backgroundProcess();
}
```

通过WebappLoader检查类文件是否更新，通过Session管理器检查是否有Session过期，通过资源管理器检查静态资源是否更新，最后调用父类ContainerBase.backgroundProcess

其中，WebappLoader通过调用Context容器的reload方法，这个方法主要做了如下事情：

- 停止、销毁Context容器及其子容器Wrapper，包括Wrapper里的Servlet
- 停止、销毁Context容器关联的Filter、Listener
- 停止、销毁Context下Pipeline和各种Valve
- 停止、销毁Context的类加载器，以及类加载器加载的类文件资源
- 启动Context容器，创建上面被销毁的资源

一个Context容器对应一个类加载器。reload方法里没有调用Session管理器的destory方法，即Context关联的Session没有销毁。热加载是默认关闭的，conf.xml设置<Context reloadable="true">开启

Tomcat的热部署和热加载有所不同。本质区别是热部署重新部署Web应用，Context对象包括关联一切资源会销毁，包括Session。通过Host容器实现，它是Context的父容器。

Host容器没有在backgroundProcess方法中实现周期性检测的任务，是通过监听器HostConfig实现的，监听周期事件

```java
public void lifecycleEvent(LifecycleEvent event) {
    // 执行check方法。
    if (event.getType().equals(Lifecycle.PERIODIC_EVENT)) {
        check();
    } 
}

protected void check() {

    if (host.getAutoDeploy()) {
        // 检查这个Host下所有已经部署的Web应用
        DeployedApplication[] apps =
            deployed.values().toArray(new DeployedApplication[0]);
            
        for (int i = 0; i < apps.length; i++) {
            //检查Web应用目录是否有变化
            checkResources(apps[i], false);
        }

        //执行部署
        deployApps();
    }
}
```

检查webapps目录下的所有Web应用，如果原来Web应用目录删掉了，就把相应的Context容器销毁，如果新的Web应用目录放进来或者WAR包放进来，就部署相应Web应用。它做的事比较宏观，不去检查具体文件或资源，只检查Web应用目录级的变化

## Context容器

### Tomcat如何打破双亲委托机制？

- JVM的类加载器

  Java的类加载，是把.class的字节码格式文件加载到JVM方法区，并在JVM的堆区建立java.lang.Class对象的实例，用来封装Java类相关的数据和方法。Class对象相当于业务类模板，JVM通过这个模板创建具体类的实例

  JVM不会在启动时加载所有类文件，而是在运行过程中用到了才会加载。而这个过程是通过类加载器来完成的。JDK提供了一个抽象类ClassLoader，如下

  ```java
  public abstract class ClassLoader {
  
      //每个类加载器都有个父加载器
      private final ClassLoader parent;
      
      public Class<?> loadClass(String name) {
    
          //查找一下这个类是不是已经加载过了
          Class<?> c = findLoadedClass(name);
          
          //如果没有加载过
          if( c == null ){
            //先委托给父加载器去加载，注意这是个递归调用
            if (parent != null) {
                c = parent.loadClass(name);
            }else {
                // 如果父加载器为空，查找Bootstrap加载器是不是加载过了
                c = findBootstrapClassOrNull(name);
            }
          }
          // 如果父加载器没加载成功，调用自己的findClass去加载
          if (c == null) {
              c = findClass(name);
          }
          
          return c；
      }
      
      protected Class<?> findClass(String name){
         //1. 根据传入的类名name，到在特定目录下去寻找类文件，把.class文件读入内存
            ...
            
         //2. 调用defineClass将字节数组转成Class对象
         return defineClass(buf, off, len)；
      }
      
      // 将字节码数组解析成一个Class对象，用native方法实现
      protected final Class<?> defineClass(byte[] b, int off, int len){
         ...
      }
  }
  ```

  该抽象类有三个默认实现类，分别是BootstrapClassLoader、ExtClassLoader、AppClassLoader，当然也可以自定义类加载器

  其中BootstrapClassLoader是启动类加载器，C实现，加载JVM启动需要的核心类，如rt.jar、resources.jar；ExtClassLoader是扩展类加载器，加载/jre/lib/ext下的jar包；AppClassLoader是系统类加载器，加载classpath下的类，应用程序默认用它加载类；自定义类加载器用来加载自定义目录下的类

  所谓双亲委派机制，即是上面代码loadClass的实现，当类没有被加载时，首先委托给父类加载器加载，层层委托之后如果还找不到，则逐层向下找子类加载器加载。要打破这个机制，重新loadClass即可。需要注意的是，虽然叫父子类，但并不是继承实现，而是通过组合实现，子类持有父类的引用。

- Tomcat的类加载器

  WebAppClassLoader是Tomcat自定义的类加载器，主要重写了loadClass和findClass,如下

  ```java
  
  public Class<?> findClass(String name) throws ClassNotFoundException {
      ...
      
      Class<?> clazz = null;
      try {
              //1. 先在Web应用目录下查找类 
              clazz = findClassInternal(name);
      }  catch (RuntimeException e) {
             throw e;
         }
      
      if (clazz == null) {
      try {
              //2. 如果在本地目录没有找到，交给父加载器去查找
              clazz = super.findClass(name);
      }  catch (RuntimeException e) {
             throw e;
         }
      
      //3. 如果父类也没找到，抛出ClassNotFoundException
      if (clazz == null) {
          throw new ClassNotFoundException(name);
       }
  
      return clazz;
  }
  ```

  其父类加载器就是系统类加载器AppClassLoader

  ```java
  public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
  
      synchronized (getClassLoadingLock(name)) {
   
          Class<?> clazz = null;
  
          //1. 先在本地cache查找该类是否已经加载过
          clazz = findLoadedClass0(name);
          if (clazz != null) {
              if (resolve)
                  resolveClass(clazz);
              return clazz;
          }
  
          //2. 从系统类加载器的cache中查找是否加载过
          clazz = findLoadedClass(name);
          if (clazz != null) {
              if (resolve)
                  resolveClass(clazz);
              return clazz;
          }
  
          // 3. 尝试用ExtClassLoader类加载器类加载，为什么？
          ClassLoader javaseLoader = getJavaseClassLoader();
          try {
              clazz = javaseLoader.loadClass(name);
              if (clazz != null) {
                  if (resolve)
                      resolveClass(clazz);
                  return clazz;
              }
          } catch (ClassNotFoundException e) {
              // Ignore
          }
  
          // 4. 尝试在本地目录搜索class并加载
          try {
              clazz = findClass(name);
              if (clazz != null) {
                  if (resolve)
                      resolveClass(clazz);
                  return clazz;
              }
          } catch (ClassNotFoundException e) {
              // Ignore
          }
  
          // 5. 尝试用系统类加载器(也就是AppClassLoader)来加载
              try {
                  clazz = Class.forName(name, false, parent);
                  if (clazz != null) {
                      if (resolve)
                          resolveClass(clazz);
                      return clazz;
                  }
              } catch (ClassNotFoundException e) {
                  // Ignore
              }
         }
      
      //6. 上述过程都加载失败，抛出异常
      throw new ClassNotFoundException(name);
  }
  ```

  第三步让ExtClassLoader加载，其目的是防止Web应用自己的类覆盖JRE的核心类。重写打破了双亲委派，此时假设Web应用自定义类Object，如果先加载它，则覆盖了JRE的Object，所以用ExtClassLoader加载，会委托给BootstrapClassLoader，发现已经加载，不会重新加载。

### Tomcat如何隔离Web应用

![image-20250114113654664](/Users/a123456/Library/Application Support/typora-user-images/image-20250114113654664.png)

Tomcat作为Servlet容器，负责加载Servlet类，此外还负责加载Servlet所依赖的jar包，并且需要加载Tomcat自己的类和依赖的jar包

1. 如果Tomcat中运行两个Web应用，都有同名的Servlet，但功能不同，Tomcat需要同时管理和加载这两个Servlet，保证不冲突。因此应用之间的类需要隔离
2. 如果两个Web应用依赖同一个三方jar包，如Spring，Tomcat要保证这两个应用可以共享Spring的jar包，只能加载一次，否则占用内存
3. Tomcat本身的类和Web应用的类也要隔离

问题1通过自定义WebAppClassLoader且给每个Web应用创建一个类加载器实例，Context容器组件对应一个Web应用，所以Context容器负责创建和维护一个WebAppClassLoader加载器实例。不同的加载器实例加载的类被任务是不同类，因为两者属于JVM中隔离的空间

问题2通过将SharedClassLoader作为WebAppClassLoader的父类，当需要加载共享的类时，委托给SharedClassLoader，解决共享的问题

问题3通过实现CatalinaClassLoader专门加载Tomcat自身的类，实现隔离

除此之外，如果Tomcat和Web应用之间有共享的类，也是一个问题。因此设计了CommonClassLoader作为CatalinaClassLoader和SharedClassLoader的父加载器处理两者的共享

4种类加载器对应的目录：CommonClassLoader-> /tomcat/common/*，CatalinaClassLoader -> /tomcat/server/*，SharedClassLoader -> /tomcat/shared/*，WebAppClassLoader -> /tomca/webapps/app/WEB-INF/*

Spring的加载问题

JVM实现中，默认如果一个类由A加载器加载，则其依赖的类也由A加载。Spring通过Class.forName加载业务类，如下

```java
public static Class<?> forName(String className) {
    Class<?> caller = Reflection.getCallerClass();
    return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
}
```

Spring作为三方jar包，其本身由SharedClassLoader加载，在这里会调用Spring的加载器去加载业务类，这时根据默认规则，SharedClassLoader会加载业务类，但是业务类又不在SharedClassLoader的加载路径下，如何解决这个问题？

通过引入线程上下文加载器解决。它是一种类加载器传递机制，类加载器保存在线程私有数据里，只要是同一个线程，一旦设置了线程上下文加载器，在线程后续执行过程中就能取出来用。Tomcat就在为每个Web应用创建WebAppClassLoader后，在启动Web应用的线程里设置线程上下文加载器，Spring启动时就会取这个加载器加载bean

线程上下文加载器是线程的私有数据，跟线程绑定，这个线程做完启动Context组件的事情后，会被回收到线程池，之后用来做其他事情，为了不影响其他事情，需要恢复之前的线程上下文加载器

```java
// SharedContext启动方法中
originalClassLoader = Thread.currentThread().getContextClassLoader();
Thread.currentThread().setContextClassLoader(webApplicationClassLoader);

// 启动方法结束时
Thread.currentThread().setContextClassLoader(originalClassLoader);
```

### Tomcat如何实现Servlet规范

加载Servlet的类不等于创建Servlet的实例，类加载只是第一步，类加载好才能创建类的实例。

一个Web应用里往往有多个Servlet，在Tomcat中一个车Web应用对应一个Context容器，即一个Context容器需要管理多个Servlet实例，但Context容器不直接持有Servlet实例，而是通过子容器Wrapper来管理Servlet的包装。Wrapper不仅仅只包含Servlet，还有相关的配置信息，如URL映射、初始化参数等。除此之外，Tomcat还需创建Listener和Filter的实例，并在合适的时机调用它们的方法

- Servlet管理

  Wrapper持有一个Servlet实例，通过loadServlet方法来实例化Servlet，这个方法主要做了两件事：创建Servlet实例，调用Servlet的init方法

  为了加快系统的启动速度，采用资源延迟加载策略，Tomcat默认启动时不会创建Servlet（配置Servlet的loadOnStartUp=true生效）。虽然启动时不创建Servlet，但是会创建Wrapper容器，当请求访问某个Servlet时，其实例才会被创建

  Tomcat每个容器组件都有自己的Pipeline，每个Pipeline中有一个Valve连，最后一个是BasicValve。Wrapper的BasicValve是StandardWrapperValve。请求到来时，逐层调用，直到Context的BasicValve会调用Wrapper的第一个Valve，最后调用到StandardWrapperValve，invoke方法如下

  ```java
  public final void invoke(Request request, Response response) {
  
      //1.实例化Servlet
      servlet = wrapper.allocate();
     
      //2.给当前请求创建一个Filter链
      ApplicationFilterChain filterChain =
          ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
  
     //3. 调用这个Filter链，Filter链中的最后一个Filter会调用Servlet
     filterChain.doFilter(request.getRequest(), response.getResponse());
  
  }
  ```

  给每个请求创建一个Filter链是因为每个请求的路径不一样，而Filter都有相应的路径映射，所以不是所有的Filter都需要处理当前的请求，根据请求的路径选择特定的Filter处理。在Filter链的最后一个Filter会调用doFilter负责调用Servlet

- Filter管理

  和Servlet一样，Filter也可以在web.xml中配置。Filter的生命周期由Context管理，启动时初始化这些Filter并保存到Map里重复使用。Filter的作用域是整个Web应用，由Wrapper管理

  Filter链的存活期很短，跟每个请求对应。一个新请求来了就动态创建一个Filter链，请求处理完，链就被回收了。ApplicationFilterFactory从Context的Map中取出Filter实例按照web.xml的配置创建特定顺序的Filter链

  ```java
  public final class ApplicationFilterChain implements FilterChain {
    
    //Filter链中有Filter数组，这个好理解
    private ApplicationFilterConfig[] filters = new ApplicationFilterConfig[0];
      
    //Filter链中的当前的调用位置
    private int pos = 0;
      
    //总共有多少了Filter
    private int n = 0;
  
    //每个Filter链对应一个Servlet，也就是它要调用的Servlet
    private Servlet servlet = null;
    
    public void doFilter(ServletRequest req, ServletResponse res) {
          internalDoFilter(request,response);
    }
     
    private void internalDoFilter(ServletRequest req,
                                  ServletResponse res){
  
      // 每个Filter链在内部维护了一个Filter数组。当前Filter位置小于Filter数组长度，说明Filter还没调完，就拿下一个调用doFilter，否则所有就都调完了，最后调Servlet的service方法
      if (pos < n) {
          ApplicationFilterConfig filterConfig = filters[pos++];
          Filter filter = filterConfig.getFilter();
  
          filter.doFilter(request, response, this);
          return;
      }
  
      servlet.service(request, response);
     
  }
  ```

  Filter本身的doFilter方法会调用Filter链的doFilter方法

  ```java
  public void doFilter(ServletRequest request, ServletResponse response,
          FilterChain chain){
          
            ...
            
            //调用Filter的方法
            chain.doFilter(request, response);
        
        }
  ```

- Listener管理

  是一种扩展机制，监听容器内部发生的事件，主要是两类

  1. 生命状态的变化，如Context容器的启动和停止、Sesssion的创建、销毁
  2. 属性的变化，如Context容器某个属性值变了、Session某个属性值变了、新请求来了等

  可在web.xml或注解方式添加监听器，在监听器里实现业务逻辑。对于Tomcat来说，它需要读取配置文件，拿到监听器类名字，实例化这些类，在合适时机调用这些监听器方法。也是在Context容器中管理的，将两类时间分开管理，分别用不同的集合存放不同的监听器

  ```java
  //监听属性值变化的监听器
  private List<Object> applicationEventListenersList = new CopyOnWriteArrayList<>();
  
  //监听生命事件的监听器
  private Object applicationLifecycleListenersObjects[] = new Object[0];
  ```

  何时触发？如Context容器的启动方法里，出发了所有的ServletContextListener

  ```java
  //1.拿到所有的生命周期监听器
  Object instances[] = getApplicationLifecycleListeners();
  
  for (int i = 0; i < instances.length; i++) {
     //2. 判断Listener的类型是不是ServletContextListener
     if (!(instances[i] instanceof ServletContextListener))
        continue;
  
     //3.触发Listener的方法
     ServletContextListener lr = (ServletContextListener) instances[i];
     lr.contextInitialized(event);
  }
  ```

  ServletContextListener是留给用户扩展的接口，监听Context的启停。ServletContextListener与Tomcat自己的声明周期事件LifecycleListener不同，后者定义在声明周期管理组件中，由基类LifecycleBase统一管理

## 新特性

### Tomcat如何支持异步Servlet

当一个新请求到达时，Tomcat和Jetty会从线程池拿出一个线程处理，这个线程调用Web应用，在Web应用处理请求的过程中，Tomcat线程会一直阻塞，知道Web应用处理完毕才输出响应，最后Tomcat才回收这个线程

当Web应用需要较长时间处理请求，Tomcat的线程一直回收不了，占用系统资源，极端下线程饥饿，没有线程处理请求。为了解决这个问题，Servlet3.0引入了异步Servlet（有超时限制，默认30s，超过则触发超时机制）。主要就是在Web应用中启动单独的线程执行耗时的请求，而Tomcat线程立即返回回收，用来处理其他请求，不等待Web应用处理请求，降低系统资源消耗，提高系统吞吐量

简单的示例

```java
@WebServlet(urlPatterns = {"/async"}, asyncSupported = true)
public class AsyncServlet extends HttpServlet {

    //Web应用线程池，用来处理异步Servlet
    ExecutorService executor = Executors.newSingleThreadExecutor();

    public void service(HttpServletRequest req, HttpServletResponse resp) {
        //1. 调用startAsync或者异步上下文
        final AsyncContext ctx = req.startAsync();

       //用线程池来执行耗时操作
        executor.execute(new Runnable() {

            @Override
            public void run() {

                //在这里做耗时的操作
                try {
                    ctx.getResponse().getWriter().println("Handling Async Servlet");
                } catch (IOException e) {}

                //3. 异步Servlet处理完了调用异步上下文的complete方法
                ctx.complete();
            }

        });
    }
}
```

1. asyncSupported=true，表明支持异步Servlet
2. Web应用调用Request对象的startAsync拿到异步上线文AsyncContext，它保存了请求和响应对象
3. Web应用开启新线程处理耗时任务，完成后调用AsyncContext.complete方法，告诉Tomcat已经处理完成

异步Servlet原理

- startAsync

  创建AsyncContext对象，保存请求的中间信息，如Request和Response对象等上下文信息。因为Tomcat的工作线程在此方法调用后直接结束回到线程池，不保存任何信息，而正在执行的请求需要地方缓存。通过它，Web应用可以处理其中的请求和响应对象，完成后将响应对象返回给浏览器

  告诉Tomcat当前的Servlet处理方法返回时，不要将响应发到浏览器，因为没处理完；且不能把对象销毁，Web应用还要使用。由于Tomcat中是CoyoteAdapter负责flush响应数据和销毁请求、响应对象，所以通过某些机制告诉CoyoteAdapter

  ```java
  this.request.getCoyoteRequest().action(ActionCode.ASYNC_START, this);
  ```

  设置Request对象的状态为异步Servlet。Connector调用CoyoteAdapter的service处理请求，而CoyoteAdapter会调用Container的service方法，当Container的service方法返回时，CoyoteAdapter判断请求是否异步Servlet请求

  ```java
  public void service(org.apache.coyote.Request req, org.apache.coyote.Response res) {
      
     //调用容器的service方法处理请求
      connector.getService().getContainer().getPipeline().
             getFirst().invoke(request, response);
     
     //如果是异步Servlet请求，仅仅设置一个标志，
     //否则说明是同步Servlet请求，就将响应数据刷到浏览器
      if (request.isAsync()) {
          async = true;
      } else {
          request.finishRequest();
          response.finishResponse();
      }
     
     //如果不是异步Servlet请求，就销毁Request对象和Response对象
      if (!async) {
          request.recycle();
          response.recycle();
      }
  }
  ```

  当CoyoteAdapter的service返回到ProtocolHandler组件时，ProtocolHandler判断返回值，如果是异步Servlet请求，把当前Socket的协议处理Processor缓存起来，将SocketWrapper对象和相应的Processor存到map里

  ```java
  private final Map<S,Processor> connections = new ConcurrentHashMap<>();
  ```

  这个请求接下来处理时，直接从map中通过SocketWrapper获取原来的Processor处理

- complete

  将相应数据发送到浏览器，这件事不能由Web应用线程做，需要Tomcat线程做。连接器中的Endpoint组件检测到有请求数据到达时，创建SocketProcessor对象交给线程池处理，因此Endpoint的通信处理和具体请求处理在两个线程运行。

  在异步Servlet的场景里，调用complete方法时，生成一个SocketProcessor任务类，交给线程池处理。相应的Socket和协议处理组件Processor都缓存了，通过Request对象拿到

  ```java 
  public void complete() {
      //检查状态合法性，我们先忽略这句
      check();
      
      //调用Request对象的action方法，其实就是通知连接器，这个异步请求处理完了
  request.getCoyoteRequest().action(ActionCode.ASYNC_COMPLETE, null);
      
  }
  
  case ASYNC_COMPLETE: {
      clearDispatches();
      if (asyncStateMachine.asyncComplete()) {
          // OPEN_READ参数可以控制ScoketProcessor的行为，不在需要把请求发给容器处理，只需要发给浏览器，然后重新在这个Socket上监听新请求
          processSocketEvent(SocketEvent.OPEN_READ, true);
      }
      break;
  }
  
  protected void processSocketEvent(SocketEvent event, boolean dispatch) {
      SocketWrapperBase<?> socketWrapper = getSocketWrapper();
      if (socketWrapper != null) {
          socketWrapper.processSocket(event, dispatch);
      }
  }
  
  public boolean processSocket(SocketWrapperBase<S> socketWrapper,
          SocketEvent event, boolean dispatch) {
          
        if (socketWrapper == null) {
            return false;
        }
        
        SocketProcessorBase<S> sc = processorCache.pop();
        if (sc == null) {
            sc = createSocketProcessor(socketWrapper, event);
        } else {
            sc.reset(socketWrapper, event);
        }
        //线程池运行
        Executor executor = getExecutor();
        if (dispatch && executor != null) {
            executor.execute(sc);
        } else {
            sc.run();
        }
  }
  ```

  ![image-20250115152345939](/Users/a123456/Library/Application Support/typora-user-images/image-20250115152345939.png)

什么场景适合异步Servlet？Tomcat线程不够，大量线程阻塞在等待Web应用处理上，而Web应用没有优化空间

### SpringBoot如何使用内嵌式Tomcat和Jetty

Tomat和Jetty是组件化设计，启动它们就是启动这些组件。Tomcat独立部署时，通过startup脚本启动，Tomcat中的Boostsrap和Catalina负责初始化类加载器，解析server.xml和启动这些组件。

在内嵌的模式下，Booststrap和Catalina的工作交给Springboot处理

SpringBoot为了支持多种Web容器，对内嵌式Web容器进行了抽象，不同的容器实现该接口

```java
public interface WebServer {
    void start() throws WebServerException;
    void stop() throws WebServerException;
    int getPort();
}
```

为了方便创建Web容器，定义了工厂接口创建WebServer

```java
public interface ServletWebServerFactory {
    WebServer getWebServer(ServletContextInitializer... initializers);
}
```

ServletContextInitializer表示ServletContext的初始化器，用于ServletContext的一些配置

```java
public interface ServletContextInitializer {
    void onStartup(ServletContext servletContext) throws ServletException;
}
```

在Web容器启动时，SpringBoot会把所有实现了ServletContextInitializer接口的类收集起来，统一调用onStartup方法。通过这个接口可以自定义自己的事情

除此以外，WebServerFactoryCustomizerBeanPostProcessor接口用于定制内嵌式Web容器。它是一个BeanPostProcessor，在postProcessBeforeInitialization过程中找Spring容器里WebServerFactoryCustomizer的bean，依次调用customize做定制化

```java
public interface WebServerFactoryCustomizer<T extends WebServerFactory> {
    void customize(T factory);
}
```

- 内嵌Web容器的创建和启动

  Spring的核心ApplicationContext，其抽象实现类AbstractApplicationContext实现refresh方法，用来创建或刷新一个ApplicationContext，其中会调用onRefresh。AbstractApplicationContext的子类可以重写onRefresh来实现特定的Context刷新逻辑，而ServletWebServerApplicationContext就是通过这个方式创建内嵌式Web容器

  ```java
  @Override
  protected void onRefresh() {
       super.onRefresh();
       try {
          //重写onRefresh方法，调用createWebServer创建和启动Tomcat
          createWebServer();
       }
       catch (Throwable ex) {
       }
  }
  
  //createWebServer的具体实现
  private void createWebServer() {
      //这里WebServer是Spring Boot抽象出来的接口，具体实现类就是不同的Web容器
      WebServer webServer = this.webServer;
      ServletContext servletContext = this.getServletContext();
      
      //如果Web容器还没创建
      if (webServer == null && servletContext == null) {
          //通过Web容器工厂来创建
          ServletWebServerFactory factory = this.getWebServerFactory();
          //注意传入了一个"SelfInitializer"
          this.webServer = factory.getWebServer(new ServletContextInitializer[]{this.getSelfInitializer()});
          
      } else if (servletContext != null) {
          try {
              this.getSelfInitializer().onStartup(servletContext);
          } catch (ServletException var4) {
            ...
          }
      }
  
      this.initPropertySources();
  }
  
  // 调用TomcatAPI创建各种组件
  public WebServer getWebServer(ServletContextInitializer... initializers) {
      //1.实例化一个Tomcat，可以理解为Server组件。
      Tomcat tomcat = new Tomcat();
      
      //2. 创建一个临时目录
      File baseDir = this.baseDirectory != null ? this.baseDirectory : this.createTempDir("tomcat");
      tomcat.setBaseDir(baseDir.getAbsolutePath());
      
      //3.初始化各种组件
      Connector connector = new Connector(this.protocol);
      tomcat.getService().addConnector(connector);
      this.customizeConnector(connector);
      tomcat.setConnector(connector);
      tomcat.getHost().setAutoDeploy(false);
      this.configureEngine(tomcat.getEngine());
      
      //4. 创建定制版的"Context"组件。Context是指Tomcat中的Context组件
      this.prepareContext(tomcat.getHost(), initializers);
      return this.getTomcatWebServer(tomcat);
  }
  ```

  为了方便控制Context组件行为，SpringBoot定制了自己的TomcatEmbeddedContext

  ```java
  class TomcatEmbeddedContext extends StandardContext {}
  ```

- 注册Servlet的方式

  1. Servlet注解：在SpringBoot启动类加@ServletComponentScan，使用@WebServlet、@WebFilter、@WebListener标记的Servlet、Filter、Listener就可以自动注册到Servlet容器中

     ```java
     @SpringBootApplication
     @ServletComponentScan
     public class xxxApplication
     {}
     
     @WebServlet("/hello")
     public class HelloServlet extends HttpServlet {}
     ```

     

  2. *RegistrationBean：Servlet、Filter、ServletListenerRegisterationBean分别用来注册Servlet、Filter、Listener

     ```java
     @Bean
     public ServletRegistrationBean servletRegistrationBean() {
         return new ServletRegistrationBean(new HelloServlet(),"/hello");
     }
     ```

  3. ServletContextInitializer：实现这个接口，并注册为一个Bean，SpringBoot就会调用它的onStartUp方法

     ```java
     @Component
     public class MyServletRegister implements ServletContextInitializer {
     
         @Override
         public void onStartup(ServletContext servletContext) {
         
             //Servlet 3.0规范新的API
             ServletRegistration myServlet = servletContext
                     .addServlet("HelloServlet", HelloServlet.class);
                     
             myServlet.addMapping("/hello");
             
             myServlet.setInitParameter("name", "Hello Servlet");
         }
     
     }
     ```

     第二种也是通过第三种实现的

- Web容器的定制

  1. ConfigurableServletWebServerFactory，用来定制一些Web容器通用的参数

     ```java
     @Component
     public class MyGeneralCustomizer implements
       WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
       
         public void customize(ConfigurableServletWebServerFactory factory) {
             factory.setPort(8081);
             factory.setContextPath("/hello");
          }
     }
     ```

     

  2. 特定的工厂：如TomcatServletWebServerFactory进一步定制，如给Tomcat增加一个Valve，向请求头加traceid，用于分布式追踪

     ```java
     class TraceValve extends ValveBase {
         @Override
         public void invoke(Request request, Response response) throws IOException, ServletException {
     
             request.getCoyoteRequest().getMimeHeaders().
             addValue("traceid").setString("1234xxxxabcd");
     
             Valve next = getNext();
             if (null == next) {
                 return;
             }
     
             next.invoke(request, response);
         }
     
     }
     
     @Component
     public class MyTomcatCustomizer implements
             WebServerFactoryCustomizer<TomcatServletWebServerFactory> {
     
         @Override
         public void customize(TomcatServletWebServerFactory factory) {
             factory.setPort(8081);
             factory.setContextPath("/hello");
             factory.addEngineValves(new TraceValve() );
     
         }
     }
     ```

## Logger组件：Tomcat的日志框架

日志模块作为一个通用的功能，在系统里通常会使用第三方的日志框架。Java的日志框架有很多，比如JUL、Log4j、Logback、Log4j2等。除此之外，还有JCL和SLF4J这样的门面日志。Logback可以说是Log4j的强化版。JCL采用运行时动态绑定机制，在运行时动态寻找和加载日志框架实现。SLF4J日志输出服务绑定比较简单，在编译时静态绑定日志框架，只需提前引入需要的日志框架

所谓门面日志是利用了设计模式中的门面模式思想，对外提供一套通用的日志记录的API，而不提供具体的日志输出，如果要实现日志输出，通过集成其他的日志框架，Log4j、Logback等。这样的好处在于记录日志的Api和日志输出的服务分离开，代码里只关注记录日志的API，通过SLF4J指定的接口记录日志，日志输出通过引入jar包指定。当需要改变日志输出服务时，无需修改代码，改变引入的jar包即可

默认情况下，Tomcat使用自身的JULI作为Tomcat内部的日志处理系统。其日志门面采用JCL，具体实现构建在Java原生的日志系统java.util.logging之上

- Java日志系统

  Java日志包在java.util.logging路径下，包含几个重要组件

  ![image-20250116141422584](/Users/a123456/Library/Application Support/typora-user-images/image-20250116141422584.png)

  Logger：记录日志的类

  Handler：规定了日志的输出方式，如控制台、写入文件

  Level：定义日志不同级别

  Formatter：将日志信息格式化，如纯文本、XML

- JULI

  JULI对日志的处理方式与Java自带基本一致，但Tomcat中可以有多个应用，而每个应用的日志应该独立。Java原生日志系统是每个JVM有一份日志的配置文件，不符合Tomcat多应用的场景，所以JULI重新实现了一些日志接口

  Log的基础实现类是DirectJDKLog，包装了一下Java的Logger类，做了一些修改。Log使用工厂模式向外提供实例，LogFactory是个单例，可以通过ServiceLoader为Log提供自定义的版本实现，没配置则使用默认的DirectJDKLog

  ```java
  private LogFactory() {
      // 通过ServiceLoader尝试加载Log的实现类
      ServiceLoader<Log> logLoader = ServiceLoader.load(Log.class);
      Constructor<? extends Log> m=null;
      
      for (Log log: logLoader) {
          Class<? extends Log> c=log.getClass();
          try {
              m=c.getConstructor(String.class);
              break;
          }
          catch (NoSuchMethodException | SecurityException e) {
              throw new Error(e);
          }
      }
      
      //如何没有定义Log的实现类，discoveredLogConstructor为null
      discoveredLogConstructor = m;
  }
  
  
  public Log getInstance(String name) throws LogConfigurationException {
      //如果discoveredLogConstructor为null，也就没有定义Log类，默认用DirectJDKLog
      if (discoveredLogConstructor == null) {
          return DirectJDKLog.getInstance(name);
      }
  
      try {
          return discoveredLogConstructor.newInstance(name);
      } catch (ReflectiveOperationException | IllegalArgumentException e) {
          throw new LogConfigurationException(e);
      }
  }
  ```

  JULI就定义了两个Handler：FileHandler和AsyncFileHandler。前者是在特定位置写文件的工具类，有一些常用写操作的方法，open、write、close、flush，使用了读写锁。日志信息通过Formatter格式化。后者继承自FileHandler，实现了异步写的操作。其中缓存存储通过LinkedBlockingDeque实现。当应用通过Handler记录一条消息时，消息会先被存在队列中，后台有一个专门的线程处理队列中的消息，去除的消息通过父类的publish方法写入相应文件内。这样当大量日志需要写入时有个缓冲，防止阻塞在写日志的动作上。可以为队列设置不同的模式，对新进入的消息又不同的处理方式，有些可能抛弃日志。

  Formatter通过format方法将日志记录LogRecord格式化成字符串。JULI有三个Formatter，OnlineFormatter：与Java自带SimpleFormatter基本一致，将内容写到一行中；VerbatimFormatter：只记录了日志信息，没有额外的信息；JdkLoggerFormatter：格式化了一个轻量级的日志信息

- 日志配置

  Tomcat的日志配置文件在Tomcat文件夹下的conf/logging.properties

  ```java
  // 数字区分同一个类的不同实例，catalina、localhost这些前缀是区分不同系统日志的标志，后面的字符串表示Handler具体类型
  handlers = 1catalina.org.apache.juli.AsyncFileHandler, 2localhost.org.apache.juli.AsyncFileHandler, 3manager.org.apache.juli.AsyncFileHandler, 4host-manager.org.apache.juli.AsyncFileHandler, java.util.logging.ConsoleHandler
  
  .handlers = 1catalina.org.apache.juli.AsyncFileHandler, java.util.logging.ConsoleHandler
  // 每个Handler设置日志等级、目录、文件前缀等
  1catalina.org.apache.juli.AsyncFileHandler.level = FINE
  1catalina.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
  1catalina.org.apache.juli.AsyncFileHandler.prefix = catalina.
  1catalina.org.apache.juli.AsyncFileHandler.maxDays = 90
  1catalina.org.apache.juli.AsyncFileHandler.encoding = UTF-8
  ```

## Manager组件：Tomcat的Session管理机制解析

可以通过Request.getSession获取Session，并通过Session读取和写入属性值。Session的管理是由Web容器完成的，主要是创建和销毁以及Session状态变化通知给监听者

也可以交给Spring管理，与特定的Web容器解耦。Spring Session的核心原理是通过Filter拦截Servlet请求，将标准的ServletRequest包装成Spring的Request，使用时Spring就会创建和管理Session

- Session创建

  Tomcat中，由每个Context容器的Manager对象管理Session，默认实现类是StandardManager

  ```java
  public interface Manager {
      public Context getContext();
      public void setContext(Context context);
      public SessionIdGenerator getSessionIdGenerator();
      public void setSessionIdGenerator(SessionIdGenerator sessionIdGenerator);
      public long getSessionCounter();
      public void setSessionCounter(long sessionCounter);
      public int getMaxActive();
      public void setMaxActive(int maxActive);
      public int getActiveSessions();
      public long getExpiredSessions();
      public void setExpiredSessions(long expiredSessions);
      public int getRejectedSessions();
      public int getSessionMaxAliveTime();
      public void setSessionMaxAliveTime(int sessionMaxAliveTime);
      public int getSessionAverageAliveTime();
      public int getSessionCreateRate();
      public int getSessionExpireRate();
      public void add(Session session);
      public void changeSessionId(Session session);
      public void changeSessionId(Session session, String newId);
      public Session createEmptySession();
      public Session createSession(String sessionId);
      public Session findSession(String id) throws IOException;
      public Session[] findSessions();
      // 从存储介质加载Session
      public void load() throws ClassNotFoundException, IOException;
      public void remove(Session session);
      public void remove(Session session, boolean update);
      public void addPropertyChangeListener(PropertyChangeListener listener)
      public void removePropertyChangeListener(PropertyChangeListener listener);
      // 将Session持久化到存储介质
      public void unload() throws IOException;
      public void backgroundProcess();
      public boolean willAttributeDistribute(String name, Object value);
  }
  ```

  当调用HttpServletRequest.getSession(true)时，true意味着如果当前请求没有Session就创建一个新的。其背后的操作是由Tomcat的HttpServletRequest实现类org.apache.catalina.connector.Request的包装类RequestFacade处理的

  ```java
  Context context = getContext();
  if (context == null) {
      return null;
  }
  
  Manager manager = context.getManager();
  if (manager == null) {
      return null;      
  }
  
  session = manager.createSession(sessionId);
  session.access();
  ```

  由上代码看出，Request对象持有Context容器对象，Context持有Manager，调用Manager.createSeesion创建Seesion。createSession是StandardManager的父类ManagerBase的默认实现

  ```java
  @Override
  public Session createSession(String sessionId) {
      //首先判断Session数量是不是到了最大值，最大Session数可以通过参数设置
      if ((maxActiveSessions >= 0) &&
              (getActiveSessions() >= maxActiveSessions)) {
          rejectedSessions++;
          throw new TooManyActiveSessionsException(
                  sm.getString("managerBase.createSession.ise"),
                  maxActiveSessions);
      }
  
      // 重用或者创建一个新的Session对象，请注意在Tomcat中就是StandardSession
      // 它是HttpSession的具体实现类，而HttpSession是Servlet规范中定义的接口
      Session session = createEmptySession();
  
  
      // 初始化新Session的值
      session.setNew(true);
      session.setValid(true);
      session.setCreationTime(System.currentTimeMillis());
      session.setMaxInactiveInterval(getContext().getSessionTimeout() * 60);
      String id = sessionId;
      if (id == null) {
          id = generateSessionId();
      }
      session.setId(id);// 这里会将Session添加到ConcurrentHashMap中
      sessionCounter++;
      
      //将创建时间添加到LinkedList中，并且把最先添加的时间移除
      //主要还是方便清理过期Session
      SessionTiming timing = new SessionTiming(session.getCreationTime(), 0);
      synchronized (sessionCreationTiming) {
          sessionCreationTiming.add(timing);
          sessionCreationTiming.poll();
      }
      return session
  }
  ```

  创建得到的Seesion被保存在ConcurrentHashMap中。Session的具体实现类是StandardSession，它实现了javax.servlet.http.HttpSession和org.apache.catalina.Session接口，然后通过StandardSessionFacade来使用，保证StandardSession的安全

  ```java
  public class StandardSession implements HttpSession, Session, Serializable {
      protected ConcurrentMap<String, Object> attributes = new ConcurrentHashMap<>();
      protected long creationTime = 0L;
      protected transient volatile boolean expiring = false;
      protected transient StandardSessionFacade facade = null;
      protected String id = null;
      protected volatile long lastAccessedTime = creationTime;
      protected transient ArrayList<SessionListener> listeners = new ArrayList<>();
      protected transient Manager manager = null;
      protected volatile int maxInactiveInterval = -1;
      protected volatile boolean isNew = false;
      protected volatile boolean isValid = false;
      protected transient Map<String, Object> notes = new Hashtable<>();
      protected transient Principal principal = null;
  }
  ```

- Session清理

  Tomcat容器组件会开启一个ContainerBackgroundProcessor后台线程，调用自己以及子容器的backgroundProcess进行一些后台逻辑处理，和Lifecycle一样，这个动作也是具有传递性的，即子容器还会把这个动作传递给自己的子容器

  ![image-20250116165838608](/Users/a123456/Library/Application Support/typora-user-images/image-20250116165838608.png)

  StandardContext重写了backgroundProcess，改为调用StandardManager的backgroundProcess完成Seesion清理工作

  ```java
  public void backgroundProcess() {
      // processExpiresFrequency 默认值为6，而backgroundProcess默认每隔10s调用一次，也就是说除了任务执行的耗时，每隔 60s 执行一次
      count = (count + 1) % processExpiresFrequency;
      if (count == 0) // 默认每隔 60s 执行一次 Session 清理
          processExpires();
  }
  
  /**
   * 单线程处理，不存在线程安全问题
   */
  public void processExpires() {
   
      // 获取所有的 Session
      Session sessions[] = findSessions();   
      int expireHere = 0 ;
      for (int i = 0; i < sessions.length; i++) {
          // Session 的过期是在isValid()方法里处理的
          if (sessions[i]!=null && !sessions[i].isValid()) {
              expireHere++;
          }
      }
  }
  ```

  默认每10s调用一次，processExpiresFrequency的作用是防止Session清理太频繁，因为遍历Session列表会耗费CPU资源

- Session事件通知

  在Servlet规范中，Session的声明周期过程需要将事件通知监听者，Servlet规范定义的Session监听器接口如下

  ```java
  public interface HttpSessionListener extends EventListener {
      //Session创建时调用
      public default void sessionCreated(HttpSessionEvent se) {
      }
      
      //Session销毁时调用
      public default void sessionDestroyed(HttpSessionEvent se) {
      }
  }
  ```

  Tomcat先创建HttpSessionEvent对象，然后遍历Context内部的LifecycleListener，判断是否HttpSessionListener实例，如果是则调用HttpSessionListener.sessionCreated进行事件通知。这些都在Seesion.setId中处理的

  ```java
  session.setId(id);
  
  @Override
  public void setId(String id, boolean notify) {
      //如果这个id已经存在，先从Manager中删除
      if ((this.id != null) && (manager != null))
          manager.remove(this);
  
      this.id = id;
  
      //添加新的Session
      if (manager != null)
          manager.add(this);
  
      //这里面完成了HttpSessionListener事件通知
      if (notify) {
          tellNew();
      }
  }
  
  public void tellNew() {
  
      // 通知org.apache.catalina.SessionListener
      fireSessionEvent(Session.SESSION_CREATED_EVENT, null);
  
      // 获取Context内部的LifecycleListener并判断是否为HttpSessionListener
      Context context = manager.getContext();
      Object listeners[] = context.getApplicationLifecycleListeners();
      if (listeners != null && listeners.length > 0) {
      
          //创建HttpSessionEvent
          HttpSessionEvent event = new HttpSessionEvent(getSession());
          for (int i = 0; i < listeners.length; i++) {
              //判断是否是HttpSessionListener
              if (!(listeners[i] instanceof HttpSessionListener))
                  continue;
                  
              HttpSessionListener listener = (HttpSessionListener) listeners[i];
              //注意这是容器内部事件
              context.fireContainerEvent("beforeSessionCreated", listener);   
              //触发Session Created 事件
              listener.sessionCreated(event);
              
              //注意这也是容器内部事件
              context.fireContainerEvent("afterSessionCreated", listener);
              
          }
      }
  }
  ```

  ![image-20250116170906976](/Users/a123456/Library/Application Support/typora-user-images/image-20250116170906976.png)

## Cluster组件：Tomcat的集群通信原理

用不太上，先不看了，大概是通过节点直接复制session，保证一致性。两种复制，朝单一节点复制和朝节点集合复制

# Jetty

jetty也是一个HTTP服务器+Servlet容器，相比Tomcat更加小巧，更易定制化

## 整体架构

![image-20241231190715251](/Users/a123456/Library/Application Support/typora-user-images/image-20241231190715251.png)

Jetty由多个Connector、多个Handler以及一个线程池组成。Connector组件实现HTTP服务器，Handler实现Servlet容器，组件工作时所需的线程资源都直接从一个全局线程池ThreadPool中获取。Connector是被所有Handler共享的，需要什么Handler，就增加什么Handler

为启动和协调三个核心组件工作，Jetty提供了一个Server类做这个事，负责创建并初始化三个组件，然后调用start方法启动

### Connector

主要负责对I/O模型和应用层协议封装。I/O模型方面，Jetty9只支持NIO。应用协议方面，抽象了Connection封装应用层协议的差异

- NIO

  - Channel：一个连接或一个Socket，通过它可以读取写入数据，不过通过Buffer来中转
  - Buffer
  - Selector：检测Channel上的I/O时间，读就绪、写就绪、链接就绪等。一个Selector可以同时处理多个Channel，即单线程可以监听多个Channel，大量减少线程上线文A线换。

  ```java
  // 创建服务端 Channel，绑定监听端口并把 Channel 设置为非阻塞方式
  ServerSocketChannel server = ServerSocketChannel.open();
  server.socket().bind(new InetSocketAddress(port));
  server.configureBlocking(false);
  // 创建 Selector，并在 Selector 中注册 Channel 感兴趣的事件 OP_ACCEPT，告诉 Selector 如果客户端有新的连接请求到这个端口就通知我
  Selector selector = Selector.open();
  server.register(selector, SelectionKey.OP_ACCEPT);
  
  // Selector 会在一个死循环里不断地调用 select 去查询 I/O 状态，select 会返回一个 SelectionKey 列表，Selector 会遍历这个列表，看看是否有“客户”感兴趣的事件，如果有，就采取相应的动作
   while (true) {
          selector.select();//查询I/O事件
          for (Iterator<SelectionKey> i = selector.selectedKeys().iterator(); i.hasNext();) { 
              SelectionKey key = i.next(); 
              i.remove(); 
  
              if (key.isAcceptable()) { 
                  // 建立一个新连接 
                  SocketChannel client = server.accept(); 
                  client.configureBlocking(false); 
                  
                  //连接建立后，告诉Selector，我现在对I/O可读事件感兴趣
                  client.register(selector, SelectionKey.OP_READ);
              } 
          }
      } 
  ```

  

Jetty设计了Acceptor、SelectorManager和Connection分别做NIO的这三件事

- Acceptor：用于接收请求。在ServerConnector中，有一个_acceptors数组，启动时创建数组长度对应数量的Acceptor，可配的。是一个Runnable，通过getExecutor获取全局线程池来执行，通过阻塞方式来接受连接。连接成功后调用accepted函数，将SocketChannel设为非阻塞，交给Selector处理

- SelectorManager：管理ManagedSelector。内部有一个数组，选一个来处理channel，并创建accept任务交给ManagedSelector。1. 调用 Selector 的 register 方法把 Channel 注册到 Selector 上，拿到一个 SelectionKey 2. 创建一个 EndPoint 和 Connection，并跟这个 SelectionKey（Channel）绑在一起。ManagedSelector没有调用Endpoint处理数据，而是将endpoint返回的runnable交给线程池执行。runnable是Endpoint的内部类，它会调用connection的回调方法处理请求

- Connection

  负责具体协议的解析，得到request对象，并调handler容器处理。

  请求处理：HttpConnection不会主动向EndPoint读取数据，而是向EndPoint注册一堆回调方法，数据到了后，回调方法会调EndPoint的接口读数据，读完后让HTTP解析器解析字节流，然后将数据封装到Request对象里

  响应处理：Connection调Handler处理请求，Handler通过Response操作响应流，向流里写入数据。然后HttpConnection再通过EndPoint吧数据写到channel，响应完成

![image-20241231193425382](/Users/a123456/Library/Application Support/typora-user-images/image-20241231193425382.png)

1. Acceptor监听连接请求，当有请求时就接受，一个请求对应一个channel，将channel交给ManagedSelector处理
2. ManagedSelector将channel注册到Selector上，创建EndPoint和Connection与channel绑定，接着不断检测I/O事件
3. I/O事件到了就调用EndPoint的方法拿到Runnable，扔给线程池执行
4. 线程池调度某个线程执行Runnable
5. Runnable执行时，调用回调函数（Connection注册到EndPoint中的）
6. 回调函数调用EndPoint的接口读取数据
7. Connection解析读到的数据，生成请求对象并交给Handler处理

### Handler

它是一个接口，有一堆实现类

```java
public interface Handler extends LifeCycle, Destroyable
{
    //处理请求的方法
    public void handle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response)
        throws IOException, ServletException;
    
    //每个Handler都关联一个Server组件，被Server管理
    public void setServer(Server server);
    public Server getServer();

    //销毁方法相关的资源
    public void destroy();
}
```

![image-20250102194127661](/Users/a123456/Library/Application Support/typora-user-images/image-20250102194127661.png)

其中AbstractHandleContainer包含其他Handler的引用，为了实现链式调用。子类HandlerWrapper包含一个Handler的引用，HandlerCollection包含一个Handler数组的引用

#### 类型

- 协调Handler：负责将请求路由到一组Handler中，如HandlerCollection内部持有一个Handler数组，负责将来到的请求转放到某个Handler
- 过滤器Handler：自己处理并转发到下一个Handler，如HandlerWrapper，内部持有一个Handler引用。其子类包含Server和ScopedHandler。Server是Handler的入口，必然将请求传递给其他Handler处理；ScopedHandler则实现了具有上下文信息的责任链调用，符合Servlet规范，执行过程中必须有上下文，可以在ScopedHandler中存储并访问
- 内容Handler：真正调用Servlet处理请求，生成响应，如ServletHandler

#### 实现Servlet规范

```java
//新建一个WebAppContext，WebAppContext是一个Handler
WebAppContext webapp = new WebAppContext();
webapp.setContextPath("/mywebapp");
webapp.setWar("mywebapp.war");

//将Handler添加到Server中去
server.setHandler(webapp);

//启动Server
server.start();
server.join();
```

WebAppContext对应Web应用。Servlet规范中有Context、Servlet、Filter、Listener、Session等。ContextHandler、ServletHandler、SessionHandler来实现规范中的功能。WebAppContext本身就是ContextHandler，且负责管理ServletHandler和SessionHandler。

ContextHandler负责创建并初始化规范里的ServletContext，且包含一组让Web应用跑起来的Handler

ServletHandler实现了Servlet、Filter、Listener功能。依赖FilterHolder、ServletHolder、ServletMapping、Filetermapping。前两个是对应的包装类，Servlet与URL的映射封装成ServletMapping，Filter和拦截URL的映射封装成FilterMapping

SessionHandler用来管理Session

除此之外还有一些通用的Handler，SecurityHandler、GzipHandler

WebAppContext将这些Handler组成一个执行链，最终会调用到执行业务的Servlet，如图

![image-20250103120812075](/Users/a123456/Library/Application Support/typora-user-images/image-20250103120812075.png)

## Jetty的线程策略EatWhatYouKill

常规NIO编程思路是：启动一个线程，在死循环中不断调用select方法，检测Channel的I/O状态，一旦I/O事件到达，就把事件和数据包装成Runnable放到新线程中处理。这样相当于是生产者-消费者关系，好处是互不干扰

Jetty的Selector编程：当Selector检测读就绪事件时，数据已经拷贝到内核中的缓存了，且CPU的缓存中也有，当应用程序读取这些数据时，如果用另一个线程读，则可能读的是另一个CPU核，这样CPU缓存中的数据就用不上，浪费了CPU高效读取且还有线程切换开销。因此，Jetty的ManagedSelector将I/O事件的侦测和处理放到同一个线程来处理，充分利用了CPU缓存并减少了线程上线文的开销。据官方测试，这种EayWhatYouKill的线程策略提高了8倍吞吐量。具体实现如下：

- ManagedSelector：封装Java的Selector并做了扩展

  ```java
  public class ManagedSelector extends ContainerLifeCycle implements Dumpable
  {
      //原子变量，表明当前的ManagedSelector是否已经启动
      private final AtomicBoolean _started = new AtomicBoolean(false);
      
      //表明是否阻塞在select调用上
      private boolean _selecting = false;
      
      //管理器的引用，SelectorManager管理若干ManagedSelector的生命周期
      private final SelectorManager _selectorManager;
      
      //ManagedSelector不止一个，为它们每人分配一个id
      private final int _id;
      
      //关键的执行策略，生产者和消费者是否在同一个线程处理由它决定
      private final ExecutionStrategy _strategy;
      
      //Java原生的Selector
      private Selector _selector;
      
      //"Selector更新任务"队列
      private Deque<SelectorUpdate> _updates = new ArrayDeque<>();
      private Deque<SelectorUpdate> _updateable = new ArrayDeque<>();
      
      ...
  }
  ```

- SelectorUpdate：更新Selector状态的抽象接口

  ```java
  /**
   * A selector update to be done when the selector has been woken.
   */
  public interface SelectorUpdate
  {
      void update(Selector selector);
  }
  ```

  对Selector的操作无非是将Channel注册到Selector或告诉Selector对什么I/O事件感兴趣，但是不能直接操作ManagedSelector中的Selector，需要向ManagedSelector提交一个任务类，即实现SelectorUpdate的update方法。执行这些update方法的是ManagedSelector自己，在死循环里拉取这些任务类逐个执行

- Selector接口

  当I/O事件到达时，ManagedSelector通过任务类接口Selectable调用对应函数处理，它返回一个Runnable，即I/O事件就绪时相应的处理逻辑

  ```java
  public interface Selectable
  {
      //当某一个Channel的I/O事件就绪后，ManagedSelector会调用的回调函数
      Runnable onSelected();
  
      //当所有事件处理完了之后ManagedSelector会调的回调函数，我们先忽略。
      void updateKey();
  }
  ```

  ManagedSelector检测到某个Channel上的I/O时间就绪时，调用这个Channel绑定的Selectable实现类的onSelected()方法拿到一个Runnable。也就是说，ManagedSelector的使用者向ManagedSelector注册事件时，也要把就绪时执行什么任务告诉它，具体就是传入一个Selectable实现类，交给线程池执行

- ExcutaionStrategy

  用于统一管理和维护用户注册的Channel集合

  ```java
  public interface ExecutionStrategy
  {
      //只在HTTP2中用到，简单起见，我们先忽略这个方法。
      public void dispatch();
  
      //实现具体执行策略，任务生产出来后可能由当前线程执行，也可能由新线程来执行
      public void produce();
      
      //任务的生产委托给Producer内部接口，
      public interface Producer
      {
          //生产一个Runnable(任务)
          Runnable produce();
      }
  }
  ```

  它将具体任务的生产委托给内部接口Producer，在自己的produce方法里实现具体逻辑，也就是生产出来的任务要么由当前线程执行，要么放到新线程中执行。Jetty提供了一些具体策略的实现类：

  - ProduceConsume：任务生产者自己生产和执行任务，即用一个线程检测和处理ManagedSelector上所有的I/O事件，一个执行完执行后一个，效率不高

    ![image-20250113174825016](/Users/a123456/Library/Application Support/typora-user-images/image-20250113174825016.png)

  - ProduceExceduteConsume：任务生产者开启新线程运行任务，不能利用CPU缓存，有现成切换成本

    ![image-20250113174844480](/Users/a123456/Library/Application Support/typora-user-images/image-20250113174844480.png)

  - ExcetueProduceConsume：任务生产者自己运行任务，且可能会新建一个新线程继续生产和执行任务。不生成自己不打算执行的任务，可以利用CPU缓存，缺点是如果I/O事件的处理时间过长，导致线程大量阻塞或线程饥饿

    ![image-20250113175104913](/Users/a123456/Library/Application Support/typora-user-images/image-20250113175104913.png)

  - EatWhatYouKill：对上一个策略的改良。在线程池充足时同上，当线程不足时，切换成ProduceExecuteComsume。因为ExcetueProduceConsume在同一线程执行I/O事件的生产和消费，其线程来自于Jetty全局的线程池。若线程被大量阻塞，甚至会导致没有检测I/O事件的线程，Connector拒绝浏览器请求。

    ```java
    private class SelectorProducer implements ExecutionStrategy.Producer
    {
        private Set<SelectionKey> _keys = Collections.emptySet();
        private Iterator<SelectionKey> _cursor = Collections.emptyIterator();
    
        @Override
        public Runnable produce()
        {
            while (true)
            {
                //如果Channel集合中有I/O事件就绪，调用前面提到的Selectable接口获取Runnable,直接返回给ExecutionStrategy去处理
                Runnable task = processSelected();
                if (task != null)
                    return task;
                
               //如果没有I/O事件就绪，就干点杂活，看看有没有客户提交了更新Selector的任务，就是上面提到的SelectorUpdate任务类。
                processUpdates();
                updateKeys();
    
               //继续执行select方法，侦测I/O就绪事件
                if (!select())
                    return null;
            }
        }
     }
    ```


## 对象池技术

Jetty使用ByteBufferPool，它是一个接口

```java
public interface ByteBufferPool
{
    // 分配内存。direct可以指定从JVM或本地内存分配
    public ByteBuffer acquire(int size, boolean direct);
		// 释放内存
    public void release(ByteBuffer buffer);
}
```

本质是一个ByteBuffer对象池，当Jetty在进行网络数据读写时，不需要每次在JVM堆上分配一块新Buffer，而是在池中获取一个预先分配好的Buffer，避免频繁分配和释放内存

```java
public class ArrayByteBufferPool implements ByteBufferPool
{
    private final int _min;//最小size的Buffer长度
    private final int _maxQueue;//Queue最大长度
    
    //用不同的Bucket(桶)来持有不同size的ByteBuffer对象,同一个桶中的ByteBuffer size是一样的
    private final ByteBufferPool.Bucket[] _direct;
    private final ByteBufferPool.Bucket[] _indirect;
    
    //ByteBuffer的size增量
    private final int _inc;
    
    public ArrayByteBufferPool(int minSize, int increment, int maxSize, int maxQueue)
    {
        //检查参数值并设置默认值
        if (minSize<=0)//ByteBuffer的最小长度
            minSize=0;
        if (increment<=0)
            increment=1024;//默认以1024递增
        if (maxSize<=0)
            maxSize=64*1024;//ByteBuffer的最大长度默认是64K
        
        //ByteBuffer的最小长度必须小于增量
        if (minSize>=increment) 
            throw new IllegalArgumentException("minSize >= increment");
            
        //最大长度必须是增量的整数倍
        if ((maxSize%increment)!=0 || increment>=maxSize)
            throw new IllegalArgumentException("increment must be a divisor of maxSize");
         
        _min=minSize;
        _inc=increment;
        
        //创建maxSize/increment个桶,包含直接内存的与heap的
        _direct=new ByteBufferPool.Bucket[maxSize/increment];
        _indirect=new ByteBufferPool.Bucket[maxSize/increment];
        _maxQueue=maxQueue;
        int size=0;
        for (int i=0;i<_direct.length;i++)
        {
          size+=_inc;
          _direct[i]=new ByteBufferPool.Bucket(this,size,_maxQueue);
          _indirect[i]=new ByteBufferPool.Bucket(this,size,_maxQueue);
        }
    }
}
```

使用不同的桶管理不同长度的ByteBuffer，其内部用ConcurrentLinkedDeque放置ByteBuffer对象的引用

![image-20250113180752290](/Users/a123456/Library/Application Support/typora-user-images/image-20250113180752290.png)

释放和分配就是找到对应的桶进行入队和出队，而不是找JVM分配和释放

## Jetty如何实现具有上下文信息的责任链

Jetty通过HandlerWrapper实现，HandlerWrapper保存下一个handler的引用，组成了一个链表。这个链表Jetty还实现了回溯的链式调用，即从头到尾依次调Handler的doScope，然后再从头到尾调Handler的doHandle。其子类ScopedHandler是很核心的一个Handler，和Servlet规范相关的Handler都直接或间接继承了它，比如ContextHandler、SessionHandler、ServletHandler、WebappHandler等

为什么要回头调一次，因为请求到达时，Jetty需要先调用各Handler的初始化方法，然后再调用各Hander的请求处理方法，且初始化必须在请求处理之前完成

- HandlerWrapper

  ```java
  public class HandlerWrapper extends AbstractHandlerContainer
  {
     protected Handler _handler;
     
     @Override
      public void handle(String target, 
                         Request baseRequest, 
                         HttpServletRequest request, 
                         HttpServletResponse response) 
                         throws IOException, ServletException
      {
          Handler handler=_handler;
          if (handler!=null)
              handler.handle(target,baseRequest, request, response);
      }
  }
  ```

  有下一个handler，有handle方法调用下一个handler

- ScopedHandler

  ```java
  // 指向当前链开头的ScopedHandler。头结点的为null
  protected ScopedHandler _outerScope;
  // 指向下一个ScopedHandler
  protected ScopedHandler _nextScope;
  
  private static final ThreadLocal<ScopedHandler> __outerScope= new ThreadLocal<ScopedHandler>();
  
  public final void handle(String target, 
                           Request baseRequest, 
                           HttpServletRequest request,
                           HttpServletResponse response) 
                           throws IOException, ServletException
  {
      if (isStarted())
      {
          if (_outerScope==null)
              doScope(target,baseRequest,request, response);
          else
              doHandle(target,baseRequest,request, response);
      }
  }
  ```

  重写了handle，通过_outerScope判断使用doScope或doHandle，这两个方法没有实现，实现了nextHandle和nextScope用来设置outScope和nextScope

  ```java
  public void doScope(String target, 
                      Request baseRequest, 
                      HttpServletRequest request, 
                      HttpServletResponse response)
         throws IOException, ServletException
  {
      nextScope(target,baseRequest,request,response);
  }
  
  public final void nextScope(String target, 
                              Request baseRequest, 
                              HttpServletRequest request,
                              HttpServletResponse response)
                              throws IOException, ServletException
  {
      if (_nextScope!=null)
          _nextScope.doScope(target,baseRequest,request, response);
      else if (_outerScope!=null)
          _outerScope.doHandle(target,baseRequest,request, response);
      else
          doHandle(target,baseRequest,request, response);
  }
  
  public abstract void doHandle(String target, 
                                Request baseRequest, 
                                HttpServletRequest request,
                                HttpServletResponse response)
         throws IOException, ServletException;
         
  
  public final void nextHandle(String target, 
                               final Request baseRequest,
                               HttpServletRequest request,
                               HttpServletResponse response) 
         throws IOException, ServletException
  {
      if (_nextScope!=null && _nextScope==_handler)
          _nextScope.doHandle(target,baseRequest,request, response);
      else if (_handler!=null)
          super.handle(target,baseRequest,request,response);
  }
  ```

  在组件启动的时候，ScopedHandler的doStart中设置_outerScope和_nextScope

  ```java
  @Override
  protected void doStart() throws Exception
  {
      try
      {
          //请注意_outScope是一个实例变量，而__outerScope是一个全局变量。先读取全局的线程私有变量__outerScope到_outerScope中
   _outerScope=__outerScope.get();
   
          //如果全局的__outerScope还没有被赋值，说明执行doStart方法的是头节点
          if (_outerScope==null)
              //handler链的头节点将自己的引用填充到__outerScope
              __outerScope.set(this);
  
          //调用父类HandlerWrapper的doStart方法
          super.doStart();
          //各Handler将自己的_nextScope指向下一个ScopedHandler
          _nextScope= getChildHandlerByClass(ScopedHandler.class);
      }
      finally
      {
          if (_outerScope==null)
              __outerScope.set(null);
      }
  }
  ```

  为什么设置全局的__outerScope，因为这个变量不能通过方法参数在Handler链中传递，而形成链的过程中又要用到它

  通过设置outScope和nextScope，并在代码中判断，目的就是让doScope在doHandle、handle方法之前执行

  ScopedHandler搭好了框架，子类只实现doScope和doHandle就行

  

  举例如下

  ```java
  ScopedHandler scopedA;
  ScopedHandler scopedB;
  HandlerWrapper wrapperX;
  ScopedHandler scopedC;
  
  scopedA.setHandler(scopedB);
  scopedB.setHandler(wrapperX);
  wrapperX.setHandler(scopedC)
  ```

  ![image-20250115180754272](/Users/a123456/Library/Application Support/typora-user-images/image-20250115180754272.png)

  调用流程如下

  ```java
  A.handle(...)
      A.doScope(...)
        B.doScope(...)
          C.doScope(...)
            A.doHandle(...)
              B.doHandle(...)
                X.handle(...)
                  C.handle(...)
                    C.doHandle(...)  
  ```

- ContextHandler

  ScopedHandler的子类，相当于Tomcat中的Context组件，对应一个Web应用，其作用是给Servlet的执行维护一个上下文环境，并将请求转发到相应的Servlet

  ```java
  private ContextHandler(Context context, HandlerContainer parent, String contextPath)
  {
      //_scontext就是Servlet规范中的ServletContext
      _scontext = context == null?new Context():context;
    
      //Web应用的初始化参数
      _initParams = new HashMap<String, String>();
      ...
  }
  
  // 主要做请求的修正、类加载器的设置，并调用nextScope
  public void doScope(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException
  {
    ...
      //1.修正请求的URL，去掉多余的'/'，或者加上'/'
      if (_compactPath)
          target = URIUtil.compactPath(target);
      if (!checkContext(target,baseRequest,response))
          return;
      if (target.length() > _contextPath.length())
      {
          if (_contextPath.length() > 1)
              target = target.substring(_contextPath.length());
          pathInfo = target;
      }
      else if (_contextPath.length() == 1)
      {
          target = URIUtil.SLASH;
          pathInfo = URIUtil.SLASH;
      }
      else
      {
          target = URIUtil.SLASH;
          pathInfo = null;
      }
  
    //2.设置当前Web应用的类加载器
    if (_classLoader != null)
    {
        current_thread = Thread.currentThread();
        old_classloader = current_thread.getContextClassLoader();
        current_thread.setContextClassLoader(_classLoader);
    }
    
    //3. 调用nextScope
    nextScope(target,baseRequest,request,response);
    
    ...
  }
  
  // 完成相应的请求
  public void doHandle(String target, Request baseRequest, HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException
  {
      final DispatcherType dispatch = baseRequest.getDispatcherType();
      final boolean new_context = baseRequest.takeNewContext();
      try
      {
           //请求的初始化工作,主要是为请求添加ServletRequestAttributeListener监听器,并将"开始处理一个新请求"这个事件通知ServletRequestListener
          if (new_context)
              requestInitialized(baseRequest,request);
  
          ...
   
          //继续调用下一个Handler，下一个Handler可能是ServletHandler、SessionHandler ...
          nextHandle(target,baseRequest,request,response);
      }
      finally
      {
          //同样一个Servlet请求处理完毕，也要通知相应的监听器
          if (new_context)
              requestDestroyed(baseRequest,request);
      }
  }
  ```

  

# Tomcat和Jetty的高性能之道

所谓高性能就是高效利用CPU、内存、网络和磁盘等资源，在短时间内处理大量的请求。衡量这个的关键指标有两个：响应时间和每秒事务处理量（TPS）

有两个资源高效利用的原则

1. 减少资源浪费：避免线程阻塞，一阻塞就会上下文切换，消耗CPU资源；网络通信时数据从内核拷贝到Java堆内存，需要在本地内存中转
2. 当某种资源成为瓶颈时，用另一种资源换取：如缓存和对象池是用内存换CPU，数据压缩后再传输是用CPU换网络

Tomcat和Jetty使用到的高性能、高并发设计

- I/O线程模型

  其本质是为了缓解CPU和外设之间的速度差。如读写网络数据，网卡数据没准备好，线程就会阻塞，让出CPU，即发生线程切换，且线程阻塞后其持有的内存没有释放，阻塞的越多，消耗的越大。因此I/O模型的目的是尽量减少线程阻塞，抛弃传统的同步阻塞I/O，采用非阻塞或异步I/O

  - 请求由专门的Acceptor线程组处理
  - I/O事件检测由专门的Selector线程组处理
  - 具体的协议解析、业务逻辑处理交给线程池，或交给Selector线程处理

  需要注意的是，并不是线程数越多越好，线程太多处理不过来，会导致大量的上下文切换

- 减少系统调用

  涉及用户态到内核态的转换。如网络通信就是系统调用最多的地方，一个Channel上的wirte就是系统调用，降低调用次数，使用缓冲，当输出数据达到一定大小才flush缓冲区

  解析HTTP协议数据时，都采用了延迟解析的策略。只有当HTTP的请求体直到用到的时候才解析。当Tomcat调用Servlet的service方法时，只读取和解析了HTTP请求头，直到Web应用程序调用了ServetRequest的getInputStream或getParameter时，才读取和解析请求体中的数据

- 池化、零拷贝

  池化是用内存换CPU，零拷贝是不做无用功，减少资源浪费

- 高效的并发编程

  并发过程中为了同步多个线程对共享变量的访问，需要加锁实现。加锁的开销较大，拿锁的过程本身就是系统调用，且如果没拿到线程会阻塞，又会发生上下文切换。当大量线程同时竞争一把锁时，会浪费大量的系统资源。

  因此，编程过程中应尽量避免锁的使用，如用原子类CAS或并发集合代替，万不得已用到锁也应缩小锁的范围和强度

  - 缩小锁范围：不直接在方法上加synchroinzed，使用细粒度的对象锁，即锁成员变量。如果在方法上直接加锁，多线程执行该方法时需要排队，在对象级别上加锁，多线程可并发执行该方法，只有访问到某成员变量才排队
  - 使用原子变量和CAS取代锁
  - 并发容器的使用：CopyOnWriteArrayList适用于读多写少的场景，如Tomcat用它存放事件监听器，因为监听器在初始化后基本不变，当事件触发是需要变量监听器列表
  - volitale关键字使用：保证可见性，一个线程修改变量，另一个线程可以读取到变化，Tomcat的LifecycleBase中用来保存声明状态



# 扩展NIO

NIO 是New IO 替代标准Java IO API。

- 区别
  - 标准IO通过字符流和字节流操作，NIO通过通道Channel和缓冲区Buffer操作，总是从Channel读到Buffer，从Buffer写入Channer
  - NIO 可以非阻塞使用，Non-blocking IO。如当线程从Channel读到Buffer时，线程还可以做其他事，同样Buffer写入Channel也一样
  - NIO引入了选择器Selectors，用于监听多个Channel的事件（到达、打开等）。单线程可以监听多个Channel

- 核心组成

  - Channels：

    - 类似于流，流是单向的，channel不是，有如下实现
      - FileChannel：从文件中读写数据
      - DatagramChannel：从UDP中读写网络中数据
      - SocketChannel：从TCP中读写网络中数据
      - ServerSocketChannel：监听新的TCP连接，每个新的都会创建一个SocketChannel

    以上涵盖了UDP、TCP网络IO，以及文件IO

    1. 如果两个channel中有一个是filechannel，则可以将数据从一个channel 传到另一个channel，通过FileChannel.transferForm()。需要注意的是，SocketChannel只会传输此刻准备好的数据（可能不足原channel的size数量）。通过FileChannel.tranerTo可以将数据从FileChannel传输到其他的channel中

    2. scatter、gather

       通常用于将传输的数据分开处理，如将消息头和消息体分到不同的buffer中，分别处理

       - scatter是在读操作时将读取的数据写入多个buffer中。需要注意的是，它不适用动态消息，大小不固定的消息，因为它是当一个buffer被填满后才会向另一个buffer写
       - gather是在写操作是将多个buffer的数据写入同一个channel。需要注意的是，write按Buffer在数组中的位置写入channel，只有position和limit之间的才会写入，这意味着它可以较好处理动态数据

  - Buffers

    - 以上涵盖了Java的基本数据类型
      - ByteBuffer
      - CharBuffer
      - DoubleBuffer
      - FloatBuffer
      - IntBuffer
      - LongBuffer
      - ShorBuffer

    - 基本用法

      1. 写数据到Buffer

      2. 调用flip

      3. 从Buffer读

      4. 调用clear\compact

      当向buffer写如数据时，buffer会记录写了多少。一旦要读，需要通过flip将buffer从写切换到读，读取之前写的所有数据。读完之后，需要清空缓冲区，让它可以再次写入，clear清空所有，compact清除已读，未读的移到缓冲的起始，新的放到后面

    - 原理

      其本质是一块可以写入、读取数据的内存，被包装成了NIO Buffer对象，提供了一组方法，方便访问

      包含三个属性：capacity、position、limit

      ![image-20250102113509156](/Users/a123456/Library/Application Support/typora-user-images/image-20250102113509156.png)

      - capacity

        容量，只能写这些个byte、long等类型数据，满了之后，清空才能继续写

      - position

        写：position表示当前位置。初始为0，写入就移到下一个位置，最大是capacity-1

        读：从position位置读，读一个移动到下一个为止。当buffer从写切换到读，position重置为0。

      - limit

        写：表示最多写多少，和capacity一样

        读：表示最多读多少。当切换成读时，limit设置成写模式的position，即能读取所有写入的数据

    - 分配

      获得一个buffer需要先分配，allocate方法

    - 写数据
      1. 从Channel写到Buffer
      2. Buffer的put方法，有不同的方式，写到指定位置，写数组等

    - 读数据

      1. 从Buffer读到channel
      2. Buffer的get方法，有不同的方式

    - 方法

      1. flip：将buffer从写切换到读，会将position设为0，limit设为之前的position

      2. rewind：将position设为0，重读buffer，limit不变

      3. clear：position设为0，limit设为capacity

      4. compact：将未读数据copy到起始，将position设到最后一个未读后面，limit设为capacity

      5. mark、reset：标记一个特定的position。之后通过reset恢复到这个position

      6. equals、：

         相同类型、剩余数据个数相同、所有剩余的元素和类型都相同，都满足两个Buffer相等

      7. compareTo

         compareTo比较两个Buffer剩余元素。第一个不相等的元素小于另一个中对应的元素，所有元素都相等，第一个数量少于第二个，都满足则buffer1<buffer2

  - Selectors

    线程->selector->channel1、channel2..

    要使用selector，先向selector注册channel，然后调用它的select()方法，此方法会阻塞到某个通道有事件就绪，一旦这个方法返回，线程就可以处理这些事件，比如新连接、数据接收等

    - 为什么使用Selector？

      系统上下文切换开销较大，且需要资源，线程越少越好。一个线程可以监听多个channel并处理

    - 创建使用

      Selector.open()创建，SelectableChannel.register()注册channel。与selector一起使用时，channel必须处于非阻塞模式下，filechannel不能切换非阻塞模式，所以不能使用，socketchannel可以。register方法的第二个参数是interest集合，可以通过|连接多个，有四种：connect：channel成功连接到另一个服务、accept：channel准备好接收新连接、read：有数据可读的channel、write：等待写数据的channel。

      SelectionKey对象，包含interest集合（selectionKey.interestOps）、ready集合(selectionKey.readyOps()或selectionKye.isAcceptable()、isConnectable、isReadable、isWritable)，channel(selectionKey.channel)，selector（selectionkey.selector），附加的对象(将一个对象或更多信息附着到selectionkey上，通过selectionkey.attach(obj)或channel.register(selector, OP_READ, obj)和selectionkey.attachment)

      注册通道后，可以调用select()方法返回感兴趣的事件，即已经就绪的channel。

      1. select() ：阻塞到至少有一个channel在你注册的事件上就绪
      2. select(long timeout)：同上，最多timeout毫秒
      3. selectNow()：不阻塞，无论什么channel就绪都立刻返回

      返回int，表示多少chanel就绪，即自上次调用后有多少channel变成就绪。

      selector不会自己从已选择的键集中移除selectionkey，处理完后需自己移除，下次它变为就绪是，会再次放入键集中

      wakeUp：某个线程调select阻塞了，即使没有channel就绪，让其他线程在第一个线程调用select方法的对象上调Selector.wakeUp方法即可

      close：关闭Selector，注册在其上的所有实例无效，通道本身不会关闭

  - pipe

    pipe有一个source通道和一个sink通道，数据被写到sink中，从source读取



# 经验

## tomcat一般生产环境线程数大小建议怎么设置呢

线程数=（(线程阻塞时间 + 线程忙绿时间) / 线程忙碌时间) * cpu核数 如果线程始终不阻塞，一直忙碌，会一直占用一个CPU核，因此可以直接设置 线程数=CPU核数。 但是现实中线程可能会被阻塞，比如等待IO。因此根据上面的公式确定线程数。























































