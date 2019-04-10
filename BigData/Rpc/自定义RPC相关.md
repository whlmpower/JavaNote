# 自定义Rpc框架

## Rpc框架server环节

### 1. 自定义注解

```Java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Component
public @interface RpcService {
    Class<?> value();
}
```

### 2. Rpc通信nettyServer

```Java
public void afterPropertiesSet() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline()
                                .addLast(new RpcDecoder(RpcRequest.class))//注册解码 IN-1
                                .addLast(new RpcEncoder(RpcResponse.class))//注册编码 OUT
                                .addLast(new RpcHandler(handlerMap)); //注册RpcHandler IN-2

                    }
                }).option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true);// 这个还不知道啥意思呢

        String[] array = serverAddress.split(":");
        String host = array[0];

        int port = Integer.parseInt(array[1]);

        ChannelFuture future = bootstrap.bind(host, port).sync();

        LOGGER.debug("server started on port {}", port);

        if (serviceRegistry != null){
            serviceRegistry.register(serverAddress);
        }
        future.channel().closeFuture().sync();
    }
```

netty中绑定消息处理流水线，包括解码、编码和业务调用

### 3. 利用Spring扫描注解，获取用户实现类

```Java
public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        Map<String, Object> serviceBeanMap = applicationContext
                .getBeansWithAnnotation(RpcService.class);

        for (Object serviceBean : serviceBeanMap.values()) {

            String interfaceName = serviceBean.getClass()
                    .getAnnotation(RpcService.class).value().getName();//获取到注解上的接口名称
            handlerMap.put(interfaceName, serviceBean);
        }
    }
```

## 用户Service环节

### 1. 引入common包，定义用户业务实现类

**添加注解RpcService**

```Java
@RpcService(HelloService.class)
public class HelloServiceImpl implements HelloService{

    @Override
    public String hello(String name) {
        System.out.println("已经调用服务端接口实现，业务处理结果为：");
        System.out.println("Hello! " + name);
        return "Hello! " + name;
    }

    @Override
    public String hello(Person person) {
        System.out.println("已经调用服务端接口实现，业务处理为：");
        System.out.println("Hello! " + person.getName() + " " + person.getAge());
        return "Hello! " + person.getName() + " " + person.getAge();
    }

}
```

### 2. 服务启动入口

```Java
/**
 * 用户系统服务端启动入口
 * 其意义在于启动springcontext，从而构造框架中的rpcServer
 * 亦即：通过注解扫描，将用户系统中标注了RpcService注解的业务发布到RpcServer中
 */
```

```Java
public class RpcBootstrap {
    public static void main(String[] args) {
        new ClassPathXmlApplicationContext("spring.xml");
    }
}

```

**步骤：**

1. 服务启动，加载Spring环境，在配置文件中构造框架RpcServer的Bean和ServiceRegister的Bean
2. RpcService 会完成注解扫描，以及netty建立，以及通过ServiceRegistry 将用户Service地址注册到Zookeeper

## 框架client环节

### 1. 构造RPC客户端，用于发送RPC请求

```Java
public RpcResponse send(RpcRequest request) throws InterruptedException {
    EventLoopGroup group = new NioEventLoopGroup();
    try{
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(group).channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel socketChannel) throws Exception {
                        socketChannel.pipeline()
                                .addLast(new RpcEncoder(RpcRequest.class)) // OUT-1
                                .addLast(new RpcDecoder(RpcResponse.class)) // IN-1
                                .addLast(RpcClient.this); //IN-2
                    }
                }).option(ChannelOption.SO_KEEPALIVE, true);

        //将request对象写入outbundle处理后发出（即RpcEncoder编码器）
        ChannelFuture future = bootstrap.connect(host, port).sync();
        // 用线程等待的方式决定是否关闭连接
        // 其意义是：先在此阻塞，等待获取到服务端的返回后，被唤醒，从而关闭网络连接
        synchronized (obj){
            obj.wait();
        }

        if (rpcResponse != null){
            future.channel().closeFuture().sync();
        }
        return rpcResponse;

    }finally {
        group.shutdownGracefully();
    }

}
```

### 2.通过反射invoke调用，利用netty发送服务请求

```Java
public <T> T create(Class<?> interfaceClass){
    return (T) Proxy.newProxyInstance(interfaceClass.getClassLoader(),
            new Class<?>[]{interfaceClass}, new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    //创建RpcRequest，封装被代理类的属性
                    RpcRequest request = new RpcRequest();
                    request.setRequestId(UUID.randomUUID().toString());
                    //查找声明这个业务方法的接口名称
                    request.setClassName(method.getDeclaringClass().getName());
                    request.setParameterTypes(method.getParameterTypes());
                    request.setParameters(args);
                    request.setMethodName(method.getName());
                    //查找服务
                    if (serviceDiscovery != null){
                        serverAddress = serviceDiscovery.discover();
                    }

                    String[] array = serverAddress.split(":");
                    String host = array[0];
                    int port = Integer.parseInt(array[1]);
                    //通过netty发送服务器请求
                    RpcClient client = new RpcClient(host, port);
                    RpcResponse response = client.send(request);
                    if (response.getError() != null){
                        throw response.getError();
                    }else {
                        return response.getResult();
                    }

                }
            });
}
```

## 用户App环节

### 1. Spring配置文件中构造RpcProxy的Bean

```xml
<bean id="serviceDiscovery" class="com.whl.rpc_registry.ServiceDiscovery">
    <constructor-arg name="registryAddress" value="${registry.address}"/>
</bean>

<bean id="rpcProxy" class="com.whl.rpc_client.RpcProxy">
    <constructor-arg name="serviceDiscovery" ref="serviceDiscovery"/>
</bean>
```

### 2. Autowired 将RpcProxy注入进来，调用create方法，实现对接口的代理，调用接口的方法，即hello方法，交给invoke实现

```Java
@Autowired
private RpcProxy rpcProxy;

@Test
public void helloTest1(){
    //调用代理的create方法，代理HelloService接口
    HelloService helloService = rpcProxy.create(HelloService.class);

    //调用代理的方法，执行invoke
    String result = helloService.hello("world");
    System.out.println("服务端返回结果：");
    System.out.println(result);
}
```

## 框架Common包

定义了编解码方法、request、response等

## 框架 Registry 和 Discovery

用户服务端注册服务的地址，用户APP端获取（可在其中实现负载均衡），需要在配置文件中进行传递对应的IP和端口号（同时netty通信也要用）。

