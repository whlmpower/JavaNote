### Bean的作用域   

Spring容器中的bean可以分为5个范围。所有范围的名称都是自说明的，但是为了避免混淆，还是让我们来解释一下：

singleton：这种bean范围是默认的，这种范围确保不管接受到多少个请求，每个容器中只有一个bean的实例，单例的模式由bean factory自身来维护。

prototype：原形范围与单例范围相反，为每一个bean请求提供一个实例。

request：在请求bean范围内会每一个来自客户端的网络请求创建一个实例，在请求完成以后，bean会失效并被垃圾回收器回收。

Session：与请求范围类似，确保每个session中有一个bean的实例，在session过期后，bean会随之失效。

global-session：global-session和Portlet应用相关。当你的应用部署在Portlet容器中工作时，它包含很多portlet。如果你想要声明让所有的portlet共用全局的存储变量的话，那么这全局变量需要存储在global-session中。

全局作用域与Servlet中的session作用域效果相同。   

### singleton 和 prototype 的区别
singleton作用域：当把一个Bean定义设置为singleton作用域是，Spring IoC容器中只会存在一个共享的Bean实例，并且所有对Bean的请求，只要id与该Bean定义相匹配，则只会返回该Bean的同一实例。值得强调的是singleton作用域是Spring中的缺省作用域。

prototype作用域：prototype作用域的Bean会导致在每次对该Bean请求（将其注入到另一个Bean中，或者以程序的方式调用容器的getBean()方法）时都会创建一个新的Bean实例。根据经验，对有状态的Bean应使用prototype作用域，而对无状态的Bean则应该使用singleton作用域。

### Spring Bean的生命周期 
![image](https://pic3.zhimg.com/80/754a34e03cfaa40008de8e2b9c1b815c_hd.jpg)  

1. Spring对Bean进行实例化（相当于程序中的new Xx()）  
2. Spring将值和Bean的引用注入进Bean对应的属性中
3. 如果Bean实现了BeanNameAware接口，Spring将Bean的ID传递给setBeanName()方法（实现BeanNameAware清主要是为了通过Bean的引用来获得Bean的ID，一般业务中是很少有用到Bean的ID的）
4. 如果Bean实现了BeanFactoryAware接口，Spring将调用setBeanDactory(BeanFactorybf)方法并把BeanFactory容器实例作为参数传入。（实现BeanFactoryAware主要目的是为了获取Spring容器，如Bean通过Spring容器发布事件等）
5. 如果Bean实现了ApplicationContextAwaer接口，Spring容器将调用setApplicationContext(ApplicationContext ctx)方法，把y应用上下文作为参数传入.(作用与BeanFactory类似都是为了获取Spring容器，不同的是Spring容器在调用setApplicationContext方法时会把它自己作为setApplicationContext的参数传入，而Spring容器在调用setBeanDactory前需要程序员自己指定（注入）setBeanDactory里的参数BeanFactory)
6. 如果Bean实现了BeanPostProcess接口，Spring将调用它们的postProcessBeforeInitialization（预初始化）方法（作用是在Bean实例创建成功后对进行增强处理，如对Bean进行修改，增加某个功能）
7. 如果Bean实现了InitializingBean接口，Spring将调用它们的afterPropertiesSet方法，作用与在配置文件中对Bean使用init-method声明初始化的作用一样，都是在Bean的全部属性设置成功后执行的初始化方法。
8. 如果Bean实现了BeanPostProcess接口，Spring将调用它们的postProcessAfterInitialization（后初始化）方法（作用与6的一样，只不过6是在Bean初始化前执行的，而这个是在Bean初始化后执行的，时机不同 )
9. 经过以上的工作后，Bean将一直驻留在应用上下文中给应用使用，直到应用上下文被销毁
10. 如果Bean实现了DispostbleBean接口，Spring将调用它的destory方法，作用与在配置文件中对Bean使用destory-method属性的作用一样，都是在Bean实例销毁前执行的方法。   

### BeanFactory 和 ApplicationContext 
BeanFactory 可以理解为含有bean集合的工厂类。BeanFactory 包含了种bean的定义，以便在接收到客户端请求时将对应的bean实例化。   
BeanFactory还能在实例化对象的时生成协作类之间的关系。此举将bean自身与bean客户端的配置中解放出来。BeanFactory还包含了bean生命周期的控制，调用客户端的初始化方法（initialization methods）和销毁方法（destruction methods）。   

从表面上看，application context如同bean factory一样具有bean定义、bean关联关系的设置，根据请求分发bean的功能。但application context在此基础上还提供了其他的功能。

提供了支持国际化的文本消息   
统一的资源文件读取方式   
已在监听器中注册的bean的事件

### Spring 配置方式
基于XML文件   
基于注解  
基于Java配置  


