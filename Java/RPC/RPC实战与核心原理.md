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


**注意事项**
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

# 高级篇
