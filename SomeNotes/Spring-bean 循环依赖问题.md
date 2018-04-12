[++参考博客++](https://blog.csdn.net/u010853261/article/details/77940767)
详细的看参考博客，这里总结和简略

Spring 中循环依赖场景  
(1) 构造器中的循环依赖
(2) field 属性中的循环依赖   

Spring解决循环依赖
在Spring容器整个生命周期中，有且只有一个对象，对象放置到Cache中，Spring为了解决单例的循环依赖问题，使用了三级缓存   
三级缓存是指：  
singletonFactories:单例对象工厂的cache   
earlySingletonObjects: 提前曝光的单例对象Cache  
singletonObjects： 单例对象cache   

循环依赖的解决所在   
A的某个field或者setter依赖于B的实例对象，B的某个field或者setter依赖于A 的实例对象，这种循环依赖的情况。A首先初始化了第一步，并将自己曝光在singletonFactories中，此时进行初始化的第二步，发现自己依赖对象B,此时去尝试获取B, 发现B没有被创建，所以B走自己的创建流程，B在初始化的第一步，发现自己依赖于对象A，尝试获取A，尝试一级缓存，二级缓存，三级缓存，由于A提前通过通过ObjectFactory提前将自己曝光了，所以B能够通过ObjectFactory拿到A的对象,虽然A没有玩去哪初始化，B拿到A的对象，顺利完成了初始化过程，然后A也完成了初始化，进入了一级缓存中，B拿到了A对象的引用，B现在也hold住的A对象完成初始化。

同时，为啥Spring不能解决A构造方法中依赖了B的实例对象，因为加入三级缓存的条件是执行了构造器，构造器的循环依赖没有完成。