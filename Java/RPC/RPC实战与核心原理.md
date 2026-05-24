# 写在前面
**为什么学**：RPC是解决分布式系统通信的利器，可以屏蔽底层的网络传输细节，让程序员专注于逻辑本身。不仅仅是用于微服务。
**如何学**：先了解基本的原理（核心流程、协议、序列化、网络通信、动态代理、实战），再学习RPC的框架（服务注册与发现、健康检测、路由策略、负载均衡、异常重试、优雅启动与关闭、熔断与限流、流量隔离），最后聊一些高级内容。

**MTThrift的一些文档**：https://km.sankuai.com/collabpage/2366230542
TODO：看完之后学习 https://km.sankuai.com/page/381359483 、https://km.sankuai.com/page/163246448 每日一粒
# 基础篇
## 核心原理
**什么是PRC**
- 从两台主机的角度：屏蔽了底层网络通信的复杂性，让程序员可以专注于业务本身。
- 从coding的角度：屏蔽了远程调用和本地调用的区别，让我们感觉就是在调用本地方法一样。
感觉划分的角度也不合适

**RPC的通信流程**
- 协议：按照协议对二进制数据进行编码，并且可以对数据格式进行标识，方便后续数据解析。
- 序列化：对象和二进制数据相互转换的过程
- 网络通信：编码成二进制数据后，两个主机就可以通过网络进行数据的传输
- 动态代理：使用Spring的AOP等技术，在方法调用处统一屏蔽底层的序列化 + 编码 + 网络通信过程。
![[Pasted image 20260221002008.png]]

**RPC在系统架构中的位置**
更像是经络，因为现在微服务之间、微服务与中间件等交互都是基于RPC来的，基于RPC才能实现现在这样庞大又复杂的系统。


**RPC使用注意事项**
1. **调用超时处理**：根据业务重要性选择策略，核心是**幂等性+降级**。见下面表格
2. **RPC使用场景**：适合**内部服务、低延迟、强一致性**场景，避免跨公网或长调用链。
3. **压缩是否开启**：权衡**数据量、带宽、CPU**，优先在高带宽消耗场景使用。因为压缩和解压也有性能消耗，需要权衡一下。
4. 在RPC的使用过程中，需要注意对象变更和序列化问题，可能会导致RPC调用失败。

| **适用场景**                | **实现方式**        | **风险/注意事项**                           |
| ----------------------- | --------------- | ------------------------------------- |
| **快速失败（Fail-Fast）**     | 非关键依赖/幂等操作      | 直接抛出异常，客户端快速感知错误                      |
| **重试（Retry）**           | 幂等操作（如读操作、支付查询） | 指数退避重试（如1s、2s、4s）+ 随机抖动               |
| **异步化（Async）**          | 非实时操作（如日志、通知）   | 将请求写入消息队列（如Kafka），后台异步处理              |
| **补偿事务（Saga）**          | 跨服务长事务（如订单+库存）  | 通过TCC/Tx模式实现正向操作+逆向补偿                 |
| **熔断（Circuit Breaker）** | 服务持续不可用         | 失败率超阈值后熔断，直接返回降级结果（如Hystrix/Sentinel） |
| **本地缓存（Cache）**         | 读多写少场景（如商品详情）   | 返回本地缓存数据（如Caffeine）                   |

## 协议
**协议的作用**：统一编码格式，方便二进制数据分割解析

**如何设计协议（定长）**：协议包含协议头和数据体。其中协议头会包含协议版本号、序列化方式，数据长度等信息。这样才能使用数据体进行解析。
![[Pasted image 20260221135358.png]]

**如何设计协议（变长）**：也需要包含协议头和数据体，只不过需要多包含一个头长度，协议头中需要增加一个变长字段，用来处理扩展型的内容。
![[Pasted image 20260221135421.png]]
**RPC如何实现request与response关联**：

HTTP1.x开启长连接，可以降低建立连接的耗时，但是仍然需要串行请求。因为HTTP无法区分两次请求的差异。性能还是比较低。

RPC有消息ID字段 + 多路复用，支持同时发起多次请求，并且在获取repsonse后，可以使用消息ID进行关联，提高了效率。

**HTTP和RPC的对比**
TODO

## 序列化
**何为序列化**：序列化就是将对象转换成二进制数据的过程，而反序列就是反过来将二进制转换为对象的过程。

**为什么需要序列化和反序列化**：因为我们程序中使用的对象，在网络传输前必须转换成二进制数据，所以就是必须的。

**常用的序列化方式**

| **序列化方式** | 序列化协议                                                                                                                                             | 优点                                                                                                                                                  | 缺点                                                                                                                                                                          |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| JDK自带序列化  | 1.  头部数据用来声明序列化协议、序列化版本，用于高低版本向后兼容<br>2.  对象数据主要包括类名、签名、属性名、属性类型及属性值，当然还有开头结尾等数据，除了属性值属于真正的对象值，其他都是为了反序列化用的元数据<br>3.  存在对象引用、继承的情况下，就是递归遍历“写对象”逻辑 |                                                                                                                                                     |                                                                                                                                                                             |
| JSON      | K-V结构的文本格式                                                                                                                                        |                                                                                                                                                     | 1. JSON进行序列化的额外空间开销比较大，对于大数据量服务这意味着需要巨大的内存和磁盘开销<br>2. JSON没有类型，如果需要反序列化成强类型对象，需要使用反射，性能都比较差。                                                                                |
| Hessian   | 动态类型、二进制、紧凑的，并且可跨语言移植<br><br>看着代码和JDK自带的好像差不多呢                                                                                                    |                                                                                                                                                     | 部分常见类型不支持<br>1. Linked系列，LinkedHashMap、LinkedHashSet等，但是可以通过扩展CollectionDeserializer类修复；<br>2. Locale类，可以通过扩展ContextSerializerFactory类修复；<br>3. Byte/Short反序列化的时候变成Integer。 |
| Protobuf  | 轻便、高效的结构化数据存储格式，可以用于结构化数据序列化<br>支持多种语言，但是需要使用不同语言的IDL进行预编译，生成序列化的工具类                                                                              | 1. 序列化后体积相比 JSON、Hessian小很多；<br>2. IDL能清晰地描述语义，所以足以帮助并保证应用程序之间的类型不会丢失，无需类似 XML 解析器；<br>3. **序列化反序列化速度很快，不需要通过反射获取类型**；<br>4. 消息格式升级和兼容性不错，可以做到向后兼容。 | 1. 不支持null；<br>2. ProtoStuff不支持单纯的Map、List集合对象，需要包在对象里面。                                                                                                                    |

- JDK自带序列化
![[Pasted image 20260221141540.png]]
- Protobuf 针对支持反射和动态能力的语言来说很费劲，可以使用ProtoStuff框架。Protostuff不需要依赖IDL文件进行预编译，可以直接对Java领域对象进行反/序列化操作，在效率上跟Protobuf差不多

**如何选择合适的序列化协议**
![[Pasted image 20260222005616.png]]
- 安全性：JDK自带的序列化有漏洞，线上服务可能就会被入侵，是最重要的考虑点。
- 通用性与兼容性：在我们进行迭代的时候，更重要的是服务的稳定性，不会出现一些不适配导致的bug～
	- 是否支持多语言、多数据类型，使用的人多不多，踩的坑多不多。

**RPC框架使用中的注意事项**
- **对象构造的不要太复杂、或者对象有复杂的继承关系**：复杂的嵌套或者继承关系会比较消耗CPU性能，影响序列化效率，而且很容易触发bug
- **对象过于庞大**：数据太大很容易造成请求超时
- **使用序列化框架不支持的类作为入参类**：序列化一般都支持原生的对象，尽量不要使用第三方对象。尤其是集合类！

## 网络通信
### 常见的网络IO模型
**同步阻塞IO（BIO）**、同步非阻塞IO（NIO）、**IO多路复用**和异步非阻塞IO（AIO）
其中只有AIO是异步IO，其他都是同步IO。
详情见[[IO模型]]

**同步阻塞IO**：
应用程序发起IO调用 -> 内核态等待数据 -> 内核态数据就绪 -> 内核态拷贝到用户态应用程序，在等待过程中，线程是完全阻塞的。

**IO多路复用**：高并发场景最常用
多个IO线程可以注册到一个复用器（select）上，会监视所有的socket，有就绪的就可以直接copy，性能比较高

**为什么上面两个更常用**：
- AIO：一般都是高版本Linux才会支持，可能是还没完全普及。
- IO多路复用：高并发场景一般都用这个
- 同步阻塞IO，一般非高并发场景都用这个
- NIO：好像IO多路复用就用这个实现的？TODO，完事儿再补充下

总的来看，这俩已经可以满足大部分需求了

**RPC一般选择哪种模型**：指定是IO多路复用

### 零拷贝
**常见的拷贝模式**：每一次调用，都需要经过CPU拷贝 + DMA拷贝两次才可以发出去，性能上有优化空间。
![[Pasted image 20260222013926.png]]

**零拷贝优化**：应用缓冲区和内核缓冲区共用一块缓冲区域，就可以减少一次拷贝～
![[Pasted image 20260222014004.png]]

PS：零拷贝未必可以减少内核态和用户态的切换次数

**零拷贝的实现方式**：这里没详细展开，有缘补充一下
1. mmap+write： 基于虚拟内存实现
2. sendfile

### Netty的零拷贝
常规零拷贝主要是解决用户空间和内核空间之间的数据拷贝。
Netty的零拷贝主要用于解决用户空间内部的数据拷贝，比如拆包发送后的包合并。

Netty也支持用户空间和内核空间之间的零拷贝，针对上面的两个实现方法也都有具体落地，这里没展开～


## 动态代理
在调用RPC方法的时候，我们感觉和使用和本地方一样便利，因为这个过程用到了动态代理。
我们实际使用只依赖方法的接口，Spring为我们注册了代理Bean，使用动态代理在内部帮我们完成了远程服务的注册和发现等能力。
![[Pasted image 20260222195607.png]]

**实现原理（以JDK自带的为例）**

``` java
/**
 * 要代理的接口
 */
public interface UserService {  
    String sayHello();  
}

/**
 * 真实调用对象
 */
public class UserServiceImpl implements UserService {  
    @Override  
    public String sayHello() {  
        return "你好";  
    }  
}

/**
 * JDK代理类生成
 */
public class JDKProxy implements InvocationHandler {
    private Object target;

    JDKProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] paramValues) {
	    // 执行前插入逻辑
         Object obj = ((RealHello)target).invoke();
        // 执行后插入逻辑
        return obj;
    }
}

/**
 * 测试例子
 */
public class TestProxy {

	public static void main(String[] args){  
	    // 构建代理器  
	    UserServiceProxy proxy = new UserServiceProxy(new UserServiceImpl());  
	    // 把生成的代理类保存到文件  
	    System.setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles","true"); 
	    // 生成代理类  
	    UserService test = (UserService) Proxy.newProxyInstance(proxy.getClass().getClassLoader(), new Class[]{UserService.class}, proxy);  
	    // 方法调用  
	    System.out.println(test.sayHello());  
}
}
```
PS：
- **InvocationHandler**：Java 动态代理（`java.lang.reflect.Proxy`）的核心接口，用于在代理对象的方法被调用时插入自定义逻辑。使用方法和代码里的一样，可以在前后插入各种各样的逻辑
- **Proxy.newProxyInstance**:
![[Pasted image 20260222211310.png]]
``` java
  public static Object newProxyInstance(ClassLoader loader,  
                                      Class<?>[] interfaces,  
                                      InvocationHandler h) throws IllegalArgumentException {
    Objects.requireNonNull(h);  
  
    final Class<?>[] intfs = interfaces.clone();  
    final SecurityManager sm = System.getSecurityManager();  
    if (sm != null) {  
        checkProxyAccess(Reflection.getCallerClass(), loader, intfs);  
    }  
  
    /*  
     * Look up or generate the designated proxy class.     */    
       Class<?> cl = getProxyClass0(loader, intfs);  
    /*  
     * Invoke its constructor with the designated invocation handler.     */   
    try {
       if (sm != null) {
	       checkNewProxyPermission(Reflection.getCallerClass(), cl);  
	    }  
	    final Constructor<?> cons = cl.getConstructor(constructorParams);  
	    final InvocationHandler ih = h;  
	    if (!Modifier.isPublic(cl.getModifiers())) {  
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
	            public Void run() {  
	                cons.setAccessible(true);  
	                return null;  
	            }  
	        });  
	    }  
	    return cons.newInstance(new Object[]{h});  
	} catch (IllegalAccessException|InstantiationException e) {  
        throw new InternalError(e.toString(), e);  
    } catch (InvocationTargetException e) {  
	    Throwable t = e.getCause();  
	    if (t instanceof RuntimeException) {  
	        throw (RuntimeException) t;  
        } else {  
	        throw new InternalError(t.toString(), t);  
	    }  
	} catch (NoSuchMethodException e) {  
        throw new InternalError(e.toString(), e);  
	}  
}
```

**动态代理选型角度**：
- **生成的字节码越小，运行所占资源就越小**：代理类是在运行中生成的，代理框架生成代理类的速度、生成代理类的字节码大小等等，都会影响到其性能。
- **生成的代理类的执行效率**：生成的代理类，是用于接口方法请求拦截的，所以每次调用接口方法的时候，都会执行生成的代理类，这个效率也很重要。
- **是否好用、易用**：API设计是否好理解、社区活跃度、还有就是依赖复杂度等等。

# 进阶篇
## RPC架构设计
**基本架构**
![[Pasted image 20260223174947.png]]
- 传输模块：默认采用TCP协议，可靠性较高。封装好了用于发送二进制数据，屏蔽底层网络差异
- 协议模块：
	- 协议包：可以规定在二进制数据流中如何插入分隔符，各方达成统一共识方便进行序列化。
	- 序列化：对象和二进制数据进行相互转换的时候，可以插入分隔符或者按照分隔符进行解析
	- 解压缩：TCP传输的时候可能会拆包发送。可以进行压缩，减少带宽压力，减少包的发送数量
- Bootstrap模块：只有传输模块和协议模块还是很复杂，需要封装一下屏蔽底层差异。
	- 动态代理：可以搞到一个bean里，通过代理生成增强bean，**这个是我们实际调用的入口**。
	- 链路追踪：请求的全链路跟踪，方便进行观察检测。
	- 过滤链：拦截请求/响应，增加鉴权、日志等功能。也可以开放扩展点，在不影响核心逻辑的情况下增强能力。
- 集群模块
	- 服务发现：在一次请求中，client和server会建立连接。连接需要双方的host和port，服务发现可以管理多方的机器信息，方便动态进行连接。
	- 连接管理：维护长连接、多路复用、生命周期管理（自动重连、心跳保活、空闲连接释放）
	- 负载均衡：在多个服务实例中**选择最优的一个**处理请求，以均衡资源使用。用于解决请求的高效分配（which to use）。
	- 路由：根据请求的**特征**（如路径、参数、Header）决定由哪个服务/实例处理。用于解决请求的正确分发（where to go）。
	- 容错：重试、熔断降级、超时控制。
	- 配置管理：动态参数配置、环境隔离等。


**扩展架构**
![[Pasted image 20260223181727.png]]
- 插件化架构：每个功能点抽象成一个接口，将这个接口作为插件的契约，然后把这个功能的接口与功能的实现分离，并提供接口的默认实现。
	- JDK有默认的SPI服务发现机制，可以自动给接口寻找实现类。但是不好用，因为不支持按需加载 + 自动装配。
- 好处：微内核
	- 扩展性好，遵循开闭原则。
	- 核心包精简，减少外部依赖，好管理。

## 服务发现

**注册中心核心问题**：
- 注册中心负载过高；（扩容、 CP）
- 各节点数据不一致；（最终一致）
- 服务下发不及时或下发错误的服务节点列表。（check + 重试）

### 为什么需要服务发现
服务发现的任务是连接服务调用方和服务提供方，让双方只有接口的时候也能get彼此的信息。
核心就俩能力：服务注册、服务订阅。
**服务注册**：服务提供方启动时，将对外暴露的接口注册保存到注册中心。
**服务订阅**：服务调用方启动时，获取服务提供方的IP，缓存到本地，用于后续RPC调用。
![[Pasted image 20260225004705.png]]


### 能否用DNS实现服务发现
不可以，因为DNS服务器为了提升性能，采取了多级缓存机制，更新频率也比较低。无法满足动态的服务注册和订阅要求。
- IP端口下线，无法及时更新，删除DNS中的IP数据
- 如果线上机器动态扩容了，无法及时把新的IP数据写到DNS服务器里
- JVM默认的DNS缓存是不会过期的

PS：DNS运行流程，获取顺序从左到右，先从JVM获取，没有就从本地缓存获取，没有就去本地DNS服务器。。。这样
![[Pasted image 20260225151706.png]]


**优化**：路由到负载均衡服务器，让服务器进行分发。
VIP看起来是一个虚拟IP服务器，可以路由到真实IP
![[Pasted image 20260225152158.png]]
**整体来看还是不方便**：
- 搭建负载均衡设备或TCP/IP四层代理，需求额外成本；
- 请求流量都经过负载均衡设备，多经过一次网络传输，会额外浪费些性能；
- 负载均衡添加节点和摘除节点，一般都要手动添加，当大批量扩容和下线时，会有大量的人工操作和生效延迟；
- 目前的负载均衡设备的分配算法不灵活，无法满足需求。

### 基于Zookeeper的服务发现（AP）
**步骤**：
1. 服务平台管理端先在ZooKeeper中创建一个服务根路径，可以根据接口名命名（例如：/service/com.demo.xxService），在这个路径再创建服务提供方目录与服务调用方目录（例如：provider、consumer），分别用来存储服务提供方的节点信息和服务调用方的节点信息。
2. 当服务提供方发起注册时，会在服务提供方目录中创建一个临时节点，节点中存储该服务提供方的注册信息。
3. 当服务调用方发起订阅时，则在服务调用方目录中创建一个临时节点，节点中存储该服务调用方的信息，同时服务调用方watch该服务的服务提供方目录（/service/com.demo.xxService/provider）中所有的服务节点数据。
4. 当服务提供方目录下有节点数据发生变更时，ZooKeeper就会通知给发起订阅的服务调用方。

**问题**：
对Zookeeper节点存储数据量过大 + 频繁读写，会导致机器状态不稳定甚至宕机。集群规模还是会有限制。

Zookeeper是强一致性，一个节点接收到数据，必须同步到其他节点才能对外提供服务，这样每个节点都是完全一致的。这就会导致集群可用性降低，
![[Pasted image 20260225155019.png]]

### 基于消息总线的最终一致性的注册中心（CP）
注册中心是可以忍受最终一致的，所以可以通过消息总线的方式实现。
每个节点保存自己的数据，不同节点使用消息总线进行数据同步，可以实现最终一致。

![[Pasted image 20260225155843.png]]
- 当有服务上线，注册中心节点收到注册请求，服务列表数据发生变化，会生成一个消息，推送给消息总线，每个消息都有整体递增的版本。
- 消息总线会主动推送消息到各个注册中心，同时注册中心也会定时拉取消息。对于获取到消息的在消息回放模块里面回放，只接受大于本地版本号的消息，小于本地版本号的消息直接丢弃，从而实现最终一致性。
- 消费者订阅可以从注册中心内存拿到指定接口的全部服务实例，并缓存到消费者的内存里面。
- 采用推拉模式，消费者可以及时地拿到服务实例增量变化情况，并和内存中的缓存数据进行合并。

推拉数据，以拉为准。


**潜在问题**
1. 在数据没完全同步的时候，可能会出现请求的节点已经下线了。在实际发生RPC之前，会进行合法性校验，如果已经下线了就重试其他节点。


## 健康检测
**为什么需要健康监测**
因为服务调用方、服务提供方、注册中心之间的网络状态瞬息万变，需要保证节点状态一直是新的、可用的。

**健康检测的逻辑 - 心跳机制**
隔30S给节点发个请求，看他能否正常正常回应。

- 初始化：机器初始化的时候，会先保持为健康状态。
- 请求 - 返回异常状态：如果探测一部分返回值异常的话，就会标记为亚健康状态。
- 请求 - 返回正常状态：如果探测结果全部正常，就可以变回健康状态。
- 请求 - 失败：节点可以直接变成死亡状态。
![[Pasted image 20260225171118.png]]
请求的时候，优先从健康状态的节点里找，没有的话就从亚健康节点列表找，如果还没有就返回失败。

**只依赖心跳机制可能存在的问题：无法及时下线异常节点**
一个节点是否健康，取决于心跳请求的返回情况。一个节点是否可以对外提供服务，取决于业务请求的响应成功率。
如果一个节点的心跳请求时好时坏，但是没有达到设立的失败阈值，就不会对他做动作。

此外，只依赖心跳机制也有可能会误伤。
1. 调用方跟服务节点之间网络状况瞬息万变，出现网络波动的时候会导致误判。
2. 在负载高情况，服务端来不及处理心跳请求，由于心跳时间很短，会导致调用方很快触发连续心跳失败而造成断开连接。

所以还是需要增加其他维度来补充一些判断依据。

**具体方案：心跳机制 + 服务维度接口可用率**
不同接口的耗时，调用次数都是不一样的，可用率是一个比较好的指标。如果可用率低于阈值了，就可以移出健康列表了

探测服务可能也有问题，可以设立多个探测节点，有一个节点正常就行～，避免误差。

## 路由策略
https://km.sankuai.com/page/461641400
路由策略就是在RPC请求的时候，把请求转发到计划的机器上。
好的路由策略可以降低服务发布期间的问题风险，这个文档只说了这个。

参数路由，根据参数情况，来选择到底路由到哪个机器上。

## 负载均衡
### 什么是负载均衡
在用集群部署服务的时候，不同节点都会承接流量，共同分摊请求压力。
负载均衡是为了保证让各个节点按照我们的预期承担应有的流量比例。

**软负载**：在单独的服务器或集群上安装负载均衡算法，LVS、Nginx，内部自行决定请求流量分配
**硬负载**：通过硬件设备实现负载均衡，如F5服务器

**负载均衡算法**：常见的有轮询、随机、最小连接

| 算法名称                       | 原理描述                     | 优点           | 缺点           | 适用场景          |
| -------------------------- | ------------------------ | ------------ | ------------ | ------------- |
| 轮询（Round Robin）            | 按顺序循环分配请求到每台服务器          | 实现简单，分配均衡    | 不考虑服务器实际负载   | 服务器性能配置一致     |
| 加权轮询（Weighted Round Robin） | 根据权重循环分配请求，权重高的服务器分配更多请求 | 适用于性能不均服务器   | 权重需人工维护      | 服务器性能有差异      |
| 随机（Random）                 | 随机选择一台服务器处理请求            | 实现简单，分配较均衡   | 有概率波动，可能短期不均 | 服务器性能相近       |
| 加权随机（Weighted Random）      | 按权重随机分配请求，权重高的概率大        | 可根据性能调整分配概率  | 权重配置复杂       | 服务器性能不均衡      |
| 最少连接（Least Connections）    | 分配给当前连接数最少的服务器           | 动态反映负载，适合长连接 | 需实时统计连接数     | 连接时长不均、负载差异大  |
| 源地址哈希（IP Hash）             | 根据客户端IP哈希分配到固定服务器        | 可实现会话保持      | 服务器变动影响分配    | 需用户粘性会话       |
| URL哈希                      | 根据请求URL哈希分配              | 适合静态资源分配     | 哈希函数需合理设计    | 路径分配需求明显      |
| 响应时间算法                     | 根据服务器平均响应时间分配，响应快的优先     | 动态调整，提升响应速度  | 需实时监控响应时间    | 对响应速度敏感业务     |
| 会话粘滞（Session Stickiness）   | 同一用户请求持续分配到同一服务器         | 保证会话状态       | 可能导致负载不均     | 电商、在线服务等需会话保持 |
| 一致性哈希（Consistent Hash）     | 请求和服务器都映射到哈希环，服务器变动影响小   | 高扩展性，适合分布式缓存 | 实现复杂         | 分布式系统、缓存等     |

### RPC中的负载均衡
RPC的负载均衡完全由RPC框架自身实现，client会和注册中心获取到的所有server建立长连接，每次PRC调用，都会根据负载均衡插件的计算结果，选择具体的服务节点。
最常用的策略就是加权随机。
![[Pasted image 20260429011208.png]]
### 自适应负载均衡策略
**有没有什么办法可以动态地、智能地控制线上服务节点所接收到的请求流量？**
负载均衡插件可以获取到所有服务节点的状态，根据状态打分后重新进行加权随机。

**根据状态打分具体是什么策略**
比如根据服务节点的负载指标、CPU核心数、内存大小，请求耗时（AVG、TP99、TP999等），服务节点状态，进行加权打分。

**具体策略**
1. 指标收集器：有运行时状态指标收集器、请求耗时指标收集器。
	1. 运行时状态指标收集器：收集服务节点CPU核数、CPU负载以及内存等指标，在服务调用者与服务提供者的心跳数据中获取。
	2. 请求耗时指标收集器：收集请求耗时数据，如平均耗时、TP99、TP999等。
2. 可以配置开启哪些指标收集器，并设置这些参考指标的指标权重，再根据指标数据和指标权重来综合打分。
3. 通过服务节点的综合打分与节点的权重，最终计算出节点的最终权重，之后服务调用者会根据随机权重的策略，来选择服务节点。
![[Pasted image 20260429011613.png]]

### MTThrift负载均衡
`effectWeight = originalWeight * preheat * W`
- effectWeight：节点实际权重
- originalWeight：节点初始权重，平台配置一般都是10
- preheat：预热系数，刚注册是0，慢慢涨到1后保持不变，差不多就是3min预热时间
- W：策略调整权重系数，下面说

`W = QPS/L`
W权重系数由LocalityAwareLoadBalancer调整，具体实现参考[[https://github.com/apache/brpc/blob/master/docs/cn/lalb.md|brpc]]。
- 目标：全局平均时延最低
- QPS：吞吐量
- L：平均时延
 
此算法可能会导致不同节点分布严重不均，因此要求W的值域在\[0.75, 1.5\],确保各个节点之间的权重不会差超出两倍。
- W越来越大：QPS一致，L越低，W越大；QPS会分配越高，直到达到性能瓶颈后L增加，W会趋于稳定
- W越来越小：QPS一致，L越大，W越小；QPS会分配越少，但是L未必下降，W会进一步下降。因为存在最小值限制，所以可能就卡死在最小的出不来了
	- 等机器正常之后，L会恢复变小，QPS就会变高，后面就会慢慢和其他节点再次达成平衡


## 异常重试
https://km.sankuai.com/page/461641868
### 为什么需要异常重试
在分布式环境中，可能因为网络抖动导致请求异常，返回失败让用户重试不够优雅。
其实内部重试一下，网络问题大概率也是可以恢复的。
因此就需要异常重试～

### RPC框架重试机制
- 是否开启重试
- 支持重试几次
- 什么样的异常需要重试：网络异常、超时等，用户自定义异常一般都不会重试
	- 可以设置白名单，业务明确可以重试的异常类型，可以识别并且主动重试
- RPC请求超时的时候，如果重试会重置超时时间，每次retry的请求也单独看配置的超时时间，不看总的（总的很容易超时）
- 如果重试的话，不让上一次执行异常的节点处理重试请求，让别人重试

**注意**：
- 幂等接口才支持重试：如果client超时，server执行了。重试就会导致执行多遍

![[Pasted image 20260508100450.png]]

### MTThrift 异常重试
Mtthrift 默认不重试，因为无法判断是否幂等

| **异常类型** | **故障特征**                    | **配置逻辑**                                 | 设计考量                                                             |     |
| -------- | --------------------------- | ---------------------------------------- | ---------------------------------------------------------------- | --- |
| **网络异常** | 瞬时性故障（如连接中断、丢包），可能通过重试自动恢复1 | 默认开启自动重试机制（如Lion配置中心的推送重试），依赖底层框架的通用重试策略 | **瞬时性**：通过重试可快速恢复，无需复杂策略。<br>**通用性**：框架层统一处理（如TCP重传机制）。          |     |
| **超时异常** | 服务端处理能力不足或长耗时操作，重试可能加剧问题    | 需显式配置独立策略（如超时时间、熔断阈值），避免级联故障             | **业务敏感性**：需结合接口幂等性、资源隔离等特性定制策略。<br>**风险控制**：避免无限重试导致雪崩（需配置熔断降级）。 |     |
网络异常需要业务自己配置单独的熔断等机制

## 优雅关闭
### 为什么需要优雅关闭
**RPC服务关闭的问题**
在分布式系统中，我们会进行服务拆分，并且单独对各个服务进行治理。这就导致了我们经常会对服务进行部署发布。

如果放任不管的话，服务调用方可能会路由到正在发布或者下线的机器，导致出现异常。
具体可能有如下两种情况：
- 调用方发请求前，目标服务已经下线。对于调用方来说，跟目标节点的连接会断开，这时候调用方可以立马感知到，并且在其健康列表里面会把这个节点挪掉，自然也就不会被负载均衡选中。
- **关键**：调用方发请求的时候，目标服务正在关闭，但调用方并不知道它正在关闭，而且两者之间的连接也没断开，所以这个节点还会存在健康列表里面，因此该节点就有一定概率会被负载均衡选中。
![[Pasted image 20260508105749.png]]

因此问题就转化成了，如何保证在下线or发布期间，不影响服务调用方

### 关闭流程
**需要做什么**：在关闭前，将即将下线的机器从Client的健康列表中移除即可。
**如何移除**：
- 人工：最稳妥，但是太墨迹，太麻烦。
- 依赖服务发现❌：通过服务发现保证最终一致性，不保证实时性。上述问题还是会存在。
	- 通过两次RPC，通知注册中心 + 服务调用方![[Pasted image 20260508111050.png]]
- 服务提供方直接通知❌：服务提供方和服务调用方本身维护有长连接，可以通过一次RPC调用直接通知，但是仍存在失败的可能。

### 优雅关闭流程
- 节点关闭时收到请求：返回特定异常（ShutdownException）给调用方，自行去其他节点重试。
	- 注册Java关闭时的钩子（ShutdownHook）：一个负责开启关闭标识，一个负责安全关闭服务对象
		- 开启关闭标识：开启挡板处理器时，新请求会判断关闭标识，已关闭会返回特定异常
		- 安全关闭服务对象：通知服务调用方该服务已关闭
- 节点关闭时处理中请求：保证可以处理完毕，如果超时处理不完，也返回特定异常让调用方重试
	- 内部使用计数器，归零后通知服务真正关闭

![[Pasted image 20260508112134.png]]

## 优雅启动
https://km.sankuai.com/page/461641788
### 为什么需要优雅启动
Java程序在刚启动的时候，执行效率会偏低。JVM会把高频代码编译成机器码，并且直接存入JVM缓存，热点代码避免重复加载 or 解释，提升执行效率。

如果放任不管的话，刚启动的机器就承担和其他线上机器一样的流量，可能会给业务带来损失。

### 启动预热

负载均衡权重根据启动时间逐渐提升，这样可以慢慢提升承接流量
- client如何获取server启动时间：这俩之间可能有点gap，不过问题不大
	- 时间一：server启动时告诉注册中心的时间
	- 时间二：注册中心收到请求并保持下来的时间
![[Pasted image 20260511131941.png]]

### 批量重启
指的是大批量重启server，没重启的server会承接很多流量，可能会有问题。
重启中的机器权重差不多，承接流量都差不多一样，没重启的权重都比较高。
可以通过[[#自适应负载均衡策略]]平缓切换

### 延迟暴露
**为什么需要延迟暴露**：
- 应用启动过程：main入口加载各种依赖类、按顺序加载SpringBean， 注册到Spring-BeanFactory，如果是RPC Bean还要注册注册中心。注册中心把server地址推给Client，Client收到地址后和Server建立长链接
- 注册好了地址，但是应用可能还没启动好，正在加载其他Bean。如果直接请求的话会报错吧

**如何避免**：注册到注册中心这一步放到最后做。
**如何实现**：
- 发布到注册中心最后做
- 发布之前整点hook，可以预加载一波资源，避免直接冷启动。
![[Pasted image 20260511132834.png]]

## 熔断限流
### 限流：服务端自我保护
单机限流、集群限流、自适应限流、黑白名单
限流算法
https://km.sankuai.com/collabpage/1246925807#b-1caecaf19e494e8481c9a8ee523387a1
### 熔断：调用端自我保护
熔断有三个状态
- Close：关闭熔断，只请求正常方法
- Open：主要请求降级方法，偶尔会尝试请求正常方法进行试探，如果试探成功的话，就会进入HALF-OPEN状态
- HALF-OPEN：半开启，一部分请求会走正常方法，一部分会走降级方法。如果请求成功的比较多，会恢复到CLOSE状态，如果请求失败的比较多，会回退到OPEN状态。
![[Pasted image 20260512003035.png]]


## 业务分组
避免其他业务流量激增， 影响我们的服务。
不同服务之间进行隔离，分而治之。
保证高可用：
- 增加每个分组的节点数量
- 提供多种分组，每个业务线有各自的分组，同时也共享一个Default分组，用于异常情况下的降级。
# 高级篇
https://km.sankuai.com/page/461641785
## 异步RPC
### 为何需要异步RPC
**提升单机吞吐量**
我们的系统大部分都是阻塞式，所以一次请求内部的多次RPC调用都会占用CPU，导致无意义空等，这会降低CPU的利用率。
提升吞吐量的关键就是提升CPU利用率，让他不白等，这就需要使用异步的技术。

## 调用端异步请求
最简单的方法就是使用Future.get方法，可以把串行的请求并行处理。


不过RPC本身内部就是异步的，通过requestId来定位唯一的请求，同步阻塞的是动态代理，代理到一定时间后get RPC的结果
![[Pasted image 20260514010143.png]]

## RPC如何异步
RPC本身支持ComputableFuture，比如定义一个返回值是ComputableFuture的接口。
Client请求Server只获取future对象，server执行完毕后，回调client的方法告诉他。

- 服务调用方发起RPC调用，直接拿到返回值CompletableFuture对象，之后就不需要任何额外的与RPC框架相关的操作了（如刚才提到的调用端异步请求，使用Future时需要通过请求上下文获取Future的操作），直接就可以进行异步处理；
- 在服务端的业务逻辑中创建一个返回值CompletableFuture对象，之后服务端真正的业务逻辑完全可以在一个线程池中异步处理，业务逻辑完成之后再调用这个CompletableFuture对象的complete方法，完成异步通知；
- 调用端在收到服务端发送过来的响应之后，RPC框架再自动地调用调用端拿到的那个返回值CompletableFuture对象的complete方法，这样一次异步调用就完成了。



## 安全体系：服务鉴权
### RPC应该关心什么安全问题
RPC一般都在内网里调用，所以很少出现数据包篡改，请求伪造等恶意行为。
一般都是不同业务对接的时候出现的坑，比如新增接口未经许可直接调用。
- 只要实现了服务调用者接口，引入服务提供者的iface，理论上都可以直接对服务进行调用。
- 已经接入服务接口了，新增调用没有报备就直接用，可能导致流量上涨，下游压力绷不住就g了

主要还是避免RPC在公司内乱调用，导致容量问题


### 如何保证调用安全：服务鉴权
**低阶方案**：每次RPC都给授权平台发个请求认证一下、或者初始化的时候鉴权好。性能平台是整体服务可用性的瓶颈，也不太好。
![[Pasted image 20260519003005.png]]

**高级点的方案**：解决集中式授权
client发请求的时候，携带自己的appkey + 加密Key发给Server。
server根据appkey判断是否有权限 + 加密key是否合法，判断是否可请求。

### 如何保证服务提供者安全：服务发现的安全问题
可能有人会自己引用了公共Jar包，自己发了一个Server去提供服务。虽然概率小但不是不可能，这样其他人就会和伪造的server交互，不安全了。

如何解决？一个接口只能有一个服务提供者，其他appkey不能发布这个接口。

不过我理解，不同appkey的接口名一样，其实也可以看作不一样的接口。
调用方不能只指定接口名，还要指定appkey。


## 分布式环境如何快速定位问题

### 分布式环境定位问题的困难
- 有问题不能debug
- 多个服务链式调用，其中一个服务报错了，直接往上抛，不好识别是哪个服务出问题了。


### 方法一：封装异常信息
AssertUtils.issuccess, 如果有报错就直接抛出去，异常信息不变。

### 方法二：分布式链路跟踪
Mtrace，一次调用会有Trace信息贯穿上下文。
根据trace可以定位到具体是哪里报错了，直接找对应服务的人排查！



## 时钟轮
### 什么是时钟轮
一种特殊的环形数组，通过槽位 + 指针定时切换槽位，可以执行定时任务。

这是一个多层时间轮代码
```java
import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

// 定时任务
class TimerTask {
    Runnable runnable;
    long expireTimeMs; // 到期时间戳

    public TimerTask(Runnable runnable, long expireTimeMs) {
        this.runnable = runnable;
        this.expireTimeMs = expireTimeMs;
    }
}

// 多层时间轮
class MultiLevelTimeWheel {
    private final int wheelSize;
    private final long tickMs;
    private final List<Queue<TimerTask>> wheel;
    private final AtomicInteger currentTick = new AtomicInteger(0);
    private final ScheduledExecutorService scheduler;
    private final MultiLevelTimeWheel higherLevel; // 上一层时间轮
    private final long startMs;

    public MultiLevelTimeWheel(int wheelSize, long tickMs, MultiLevelTimeWheel higherLevel) {
        this.wheelSize = wheelSize;
        this.tickMs = tickMs;
        this.higherLevel = higherLevel;
        this.wheel = new ArrayList<>(wheelSize);
        for (int i = 0; i < wheelSize; i++) {
            wheel.add(new ConcurrentLinkedQueue<>());
        }
        this.scheduler = Executors.newSingleThreadScheduledExecutor();
        this.startMs = System.currentTimeMillis();
    }

    public void start() {
        scheduler.scheduleAtFixedRate(this::tick, tickMs, tickMs, TimeUnit.MILLISECONDS);
        if (higherLevel != null) higherLevel.start();
    }

    public void stop() {
        scheduler.shutdown();
        if (higherLevel != null) higherLevel.stop();
    }

    // 添加任务
    public void addTask(Runnable task, long delayMs) {
        long expireTimeMs = System.currentTimeMillis() + delayMs;
        addTask(new TimerTask(task, expireTimeMs));
    }

    private void addTask(TimerTask timerTask) {
        long delayMs = timerTask.expireTimeMs - System.currentTimeMillis();
        if (delayMs < 0) delayMs = 0;
        long maxDelay = wheelSize * tickMs;
        if (delayMs < maxDelay) {
            // 本层可管理，放入对应槽位
            int ticks = (int)(delayMs / tickMs);
            int slot = (currentTick.get() + ticks) % wheelSize;
            wheel.get(slot).add(timerTask);
        } else if (higherLevel != null) {
            // 超本层，递归放到高层
            higherLevel.addTask(timerTask);
        } else {
            // 超最大范围，放到最后一个槽
            wheel.get(wheelSize - 1).add(timerTask);
        }
    }

    private void tick() {
        int tick = currentTick.getAndIncrement() % wheelSize;
        Queue<TimerTask> tasks = wheel.get(tick);
        Iterator<TimerTask> iter = tasks.iterator();
        while (iter.hasNext()) {
            TimerTask t = iter.next();
            long now = System.currentTimeMillis();
            if (t.expireTimeMs <= now) {
                try {
                    t.runnable.run();
                } catch (Exception e) {
                    e.printStackTrace();
                }
                iter.remove();
            } else {
                // 未到期，重新加入到本层或高层
                iter.remove();
                addTask(t);
            }
        }
    }

    // 示例main方法
    public static void main(String[] args) throws InterruptedException {
        // 第二层：60槽，每tick 1秒，最大范围60秒
        MultiLevelTimeWheel level2 = new MultiLevelTimeWheel(60, 1000, null);
        // 第一层：60槽，每tick 60秒，最大范围3600秒（1小时），管理更长任务
        MultiLevelTimeWheel level1 = new MultiLevelTimeWheel(60, 60_000, level2);

        level1.start();

        // 5秒后执行
        level1.addTask(() -> System.out.println("任务1执行，5秒后"), 5000);
        // 70秒后执行（会先放到level1，再转到level2）
        level1.addTask(() -> System.out.println("任务2执行，70秒后"), 70_000);
        // 1小时后执行（会一直在level1）
        level1.addTask(() -> System.out.println("任务3执行，1小时后"), 3_600_000);

        Thread.sleep(75_000);
        level1.stop();
    }
}
```


### 为什么需要时钟轮
最low的定时任务，就是sleep，期间会一直占用CPU进行处理。很浪费CPU性能。
时钟轮就可以解决这个问题，等到你了再去执行，没到你就不用管。

### 时钟轮在RPC的应用
- 定时任务
- 快速部署启动，定时1min没启动好就算失败
- 心跳机制，10S跳一次，执行一次后创建一个新的再丢进去

**注意**，时钟轮存在的问题
- 时间槽位的单位时间越短，时间轮触发任务的时间就越精确：例如时间槽位的单位时间是10毫秒，那么执行定时任务的时间误差就在10毫秒内，如果是100毫秒，那么误差就在100毫秒内。

- 时间轮的槽位越多，那么一个任务被重复扫描的概率就越小，因为只有在多层时钟轮中的任务才会被重复扫描。比如一个时间轮的槽位有1000个，一个槽位的单位时间是10毫秒，那么下一层时间轮的一个槽位的单位时间就是10秒，超过10秒的定时任务会被放到下一层时间轮中，也就是只有超过10秒的定时任务会被扫描遍历两次，但如果槽位是10个，那么超过100毫秒的任务，就会被扫描遍历两次
	- 这个我理解就是，重复扫描的话可能也会占用CPU吧？



## 流量回放
**什么是流量回放**：录制请求request + response，用来验证新逻辑是否符合预期，一般比QA的用例覆盖的更全面。

**如何录制流量**：拦截RPC请求，异步记录下来就行了。后面可以再直接模拟调用进行回放。


## 动态分组
**什么是动态分组**：感觉就是弹性扩容，划分好的分组预留一定机器冗余后，在高峰期仍然有可能会不够用，可能会影响业务稳定性。此时可以通过机器池 + 动态划分分组的形式，快速支持到各个业务分组。

不过这里说的动态分组，指的是一类服务的分组的冗余机器，可以临时划分到不同LiteSet，不用部署。
如果是纯公共池机器的话，我理解还是要重新部署发布的。


## 泛化调用：没有接口时也可以发起RPC调用
RPC的Server和Client都是通过动态代理去交互请求的，实际上我们只需要能直接给Server发动态代理处理好的消息就行了，里面需要按照格式发送下图中的相关信息。
![[Pasted image 20260522004419.png]]


RPC框架一般都支持GenericService，内部会通过反射自动帮创建对应的方法名，如果没有的话是调不通的
此外，需要指定appkey，才能正确路由到目标的服务分组
这里也有讲解 
https://cloud.tencent.com/developer/article/2364374



## RPC支持多协议

### 为什么要支持多协议
RPC技术有很多，我们的技术栈可能需要不断迭代来适配新的需求，在迭代过程中可能会遇到需要切换协议的情况（但我感觉大概率不会）




# RPC框架实例详解
## 服务端创建流程
1. 根据ProviderConfig的配置信息生成registryUrl（注册中心URL对象）与serviceUrl（服务URL对象）；
2. 根据registryUrl，调用Registry插件，创建Registry对象，Registry对象为注册中心对象，与注册中心进行交互；
3. 调用Registry对象的open方法，开启注册中心对象，也就是与注册中心建立连接；
4. 调用Registry对象的subscribe方法，订阅接口的配置信息与全局配置信息；
5. 调用InvokerManager，创建Exporter对象；
6. InvokerManager返回Exporter对象。
![[Pasted image 20260524230211.png]]


## 服务端开启流程
1. 调用Exporter对象的open方法，开启服务端；
2. Exporter对象调用接口预热插件，进行接口预热；
3. Exporter对象调用传输层中的EndpointFactroy插件，创建一个Server对象，一个Server对象就代表一个端口了；
4. 调用Server对象的open方法，开启端口，端口开启之后，服务端就可以提供远程服务了；
5. Exporter对象调用Registry对象的register方法，将这个调用端节点注册到注册中心中。
![[Pasted image 20260524230255.png]]


## 调用端启动流程
1. 根据ConsumerConfig的配置信息生成registryUrl（注册中心URL对象）与serviceUrl（服务URL对象）；
2. 根据registryUrl，调用Registry插件，创建Registry对象，Registry对象为注册中心对象，与注册中心进行交互；
3. 创建动态代理对象；
4. 调用Registry对象的Open方法，开启注册中心对象；
5. 调用Registry对象subscribe方法，订阅接口的配置信息与全局配置信息；
6. 调用InvokeManager的refer方法，用来创建Refer对象；
7. InvokeManager在创建Refer对象之前会先创建Cluster对象，Cluser对象是集群层的核心对象，Cluster会维护该调用端与服务端节点的连接状态；
8. InvokeManager创建Refer对象；
9. Refer对象初始化，其中主要包括创建路由策略、消息分发策略、创建负载均衡、调用链、添加eventbus事件监听等等；
10. ConsumerConfig调用Refer的open方法，开启调用端；
11. Refer对象调用Cluster对象的open方法，开启集群；
12. Cluster对象调用Registry对象的subcribe方法，订阅服务端节点变化，收到服务端节点变化时，Cluster会调用传输层EndpointFactroy插件，创建Client对象，与这些服务节点建立连接，Cluster会维护这些连接；
13. ConsumerConfig调用Refer对象封装到ConsumerInvokerHandler中，将ConsumerInvokerHandler对象注入给动态代理对象。
![[Pasted image 20260524230323.png]]


## RPC调用流程

### 调用端发送流程

![[Pasted image 20260524230354.png]]
1. 动态代理对象调用ConsumerInvokerHandler对象的Invoke方法；
2. ConsumerInvokerHandler对象生成请求消息对象；
3. ConsumerInvokerHandler对象调用Refer对象的Invoke方法；
4. Refer对象对请求消息对象进行处理，如设置接口信息、分组信息等等；
5. Refer对象调用消息透传插件，处理透传信息，其中就包括隐式参数信息；
6. Refer对象调用FilterChain对象的Invoker方法，执行调用链；
7. FilterChain对象调用每个Filter；
8. Refer对象的distribute方法作为最后一个Filter，被调用链最后一个执行。
9. 调用NodeSelecter对象的select方法，NodeSelecter是集群层的路由规则节点选择器，其select方法用来选择出符合路由规则的服务节点；
10. 调用Route对象的route方法，Route对象为路由分发器，也是集群层中的对象，默认为路由分发策略为Failover，即请求失败后可以重试请求，这里你可以回顾下[[第 12 讲]](https://time.geekbang.org/column/article/211261)，在这一讲的思考题中我就问过异常重试发送在RPC调用中的哪个环节，其实就在此环节；
11. Route对象调用LoadBalance对象的select方法，通过负载均衡选择一个节点；
12. Route对象回调Refer对象的invokeRemote方法；
13. Refer对象的invokeRemote方法调用传输层中Client对象，向服务端节点发送消息。

### 服务端接收流程
1. 传输层接收到请求，触发协议适配器ProtocolAdapter；
2. ProtocolAdapter对象遍历Protocol插件的实现类，匹配协议；
3. 匹配协议之后，根据Protocol对象，传输层的Server对象绑定该协议的编解码器（Codec对象）、Channel处理链（ChainChannelHandler对象）；
4. 对接收的消息进行解码与反序列化；
5. 执行Channel处理链；
6. 在业务线程池中调用消息处理链（MessageHandle插件）；
7. 调用BizReqHandle对象的handle方法，处理请求消息；
8. BizReqHandle对象调用restore方法，根据连接Session信息，处理请求消息数据，并根据请求的接口名、分组名与方法名，获取Exporter对象；
9. 调用Exporter对象的invoke方法，Exporter对象返回CompletableFuture对象；
10. Exporter对象调用FilterChain的invoke方法；
11. FilterChain执行所有Filter对象；
12. Exporter对象的invokeMethod方法作为最后一个Filter，最后被调用；
13. Exporter对象的invokeMethod方法处理请求上下文，执行反射；
14. Exporter对象将执行反射之后得到的请求结果异步通知给BizReqHandle对象；
15. BizReqHandle调用传输层的Channel对象，发送响应结果；
16. 传输层对响应消息进行协议转换、序列化、编码，最后通过网络传输响应给调用端。
![[Pasted image 20260524230413.png]]

