```java
CompletableFuture[] cfs = taskList.stream().map(integer -> CompletableFuture.supplyAsync(() -> cal(integer), executorService)

.thenApply(h -> Integer.toString(h))

.whenComplete((s, e) -> 

{System.out.println(s +  e);

 list.add(s)})).toArray(CompleteFuture[] :: new);

CompletableFuture.allOf(cfs).join();
```

## Java 8实战中CompletableFuture 

**提供异步API**

**将同步代码变成非阻塞代码**

**两个异步操作合并为一个异步操作**

**响应式的方式处理异步操作的完成事件**

#### 使用工厂方法supplyAsync 创建CompletableFuture

```Java
public Future<Double> getPriceAsync(String product){
    return CompletableFuture.supplyAsync(() -> calculatePrice(Product));
}
```

supplyAsync 方法接受一个生产者（supplier）作为参数，返回一个CompletableFuture对象，该对象完成异步执行后会读取调用生产者方法的返回值，生产者方法会交给ForkJoinPool池中某个执行线程运行。重载版本，传递第二个参数指定不同的执行线程执行生产者方法。

#### 使用并行流对请求进行并行操作

```Java
public List<String> findPrices(String product){
    return shops.parallelStream()
        .map(shop -> String.format("%s price is %.2f", shop.getName(), 
                                  shop.getPrice(product)))
        .collet(toList());
}
```

#### 使用CompletableFuture 发起异步请求

```Java
public List<String> findPrices(){
	List<CompletableFuture<String>> priceFutures = 
    	shops.stream()
    	.map(shop -> CompletableFuture.supplyAsync(
    	() -> String.format("%s price is %.2f", 
                       shop.getName(), shop.getPrice(product))))
    	.collect(toList());
    return priceFutures.stream()
        .map(CompletableFuture::join)
        .collet(toList());
}

```

对List中所有future对象进行join操作，一个接一个地等待他们运行结束，注意CompletableFuture类中join方法和Future接口中的get有相同的含义，唯一的不同是join不会抛出任何检测到的异常。

**CompletableFuture 版本的程序似乎比并行流版本的程序快一点儿，究其原因：内部采用的是同样的通用线程池，默认都使用固定数目的线程，具体线程数取决于Runtime.getRuntime().availableProcessors()的返回值，然而CompletableFuture具有一定的优势，因为它允许你对执行器进行配置，尤其是线程池的大小**

#### 使用定制的执行器

线程池大小与处理器利用率之比可以使用下面的公式进行估算：

```JSON
N(threads) = N(cpu) * U(cpu) *(1 + W/c)
// N(cpu) 是处理器的核的数目，通过Runtime.getRuntime().availableProcessors()
// U(cpu) 是期望的CPU利用率
// W/C 是等待时间与计算时间的比率
```

```Java
private final Executor executor = 
    Executors.newFixedThreadPool(Math.min(shops.size(), 100), 
                                 new ThreadFactory(){
                                     public Thread newThread(Runnable r){
                                         Thread t = new Thread(r);
                                         t.setDaemon(true);
                                         return t;
                                     }
                                 });
```

创建一个由守护线程构成的线程池，Java程序无法终止或退出一个正在运行的线程，所以最后剩下的那个线程会由于一直等待无法发生的事件而引发问题。如果将线程标记为守护线程，意味着程序退出时它也会被回收。

#### 使用并行流还是CompletableFuture

集合进行并行计算有两种方式：

1. 转化为并行流
2. 枚举集合每一个元素，创建新的线程，在CompletableFuture内对其进行操作

使用建议：计算密集型的操作，并且没有IO，推荐使用Stream接口（计算密集型，没必要创建比处理器核数更多的线程）；设计IO操作，使用CompletableFuture灵活性更好。

#### 构造同步和异步操作

```Java
public List<String> findPrices(String product){
    List<CompletableFuture<String>> priceFutures = 
        shops.stream()
        	.map(shop -> CompletableFuture.supplyAsync(
            				() -> shop.getPrice(product), executor))
        	.map(future -> future.thenApply(Quote::parse))
        	.map(future -> future.thenCompose(quote -> 
                                             CompletableFuture.supplyAsync(
                                      () -> Discount.applyDiscount(quote), executor)))
        	.collect(toList());
    
    return priceFutures.stream()
        			.map(CompletableFuture::join)
        			.collect(toList());
    
}
```

第二次转换将字符转变成订单，由于一般情况下解析操作不涉及任何远程服务，也不会进行任何IO，几乎在第一时间进行，所以能够采用同步操作，不会带来太多的延迟。使用thenApply，将一个字符串转换Quote的方法作为参数传递给它。

第三个map操作涉及远程的Discount服务，选择supplyAsync工厂方法。

thenCompose 方法允许对两个异步操作进行流水线，第一个操作完成时，将其结果作为参数传递给第二个操作。

#### 将两个CompletableFuture对象整合起来

thenCombine，接收名为BiFunctionde 的第二个参数，这个参数定义了当两个CompletableFuture对象完成计算后，如何进行合并。

```Java
Future<Double> futurePriceInUSD = 
    	CompletableFuture.supplyAsync(() -> shop.getPrice(product))
    	.thenCombine(
			CompletableFuture.supplyAsync(
            () -> exchangeService.getRate(Money.EUR, Money.USD)),
    		(price, rate) -> price*rate
		);
```

#### 响应CompletableFuture的Completion事件

新添加的操作会在CompletableFuture完成执行后使用它的返回值。thenAccept 方法提供了这一功能，接收CompletableFuture执行完毕后的返回值做参数。

一旦CompletableFuture 计算得到结果，返回一个CompletableFuture<Void> ，map 操作返回的是一个stream<CompletableFuture<Void>> ，对这个对象，能做的事情非常有限，只能等待其运行结束。你还希望能给最慢的商店一些机会，让他们有机会打印输出返回的价格。为了实现这个目的， 把构成Stream的所有的CompletableFuture<Void>对象放到一个数组中，等待所有任务执行完成。

```Java
findPricesStream("myPhone").map(f -> f.thenAccept(System.out::println));
```

```Java
CompletableFuture[] futures = findPricesStream("myPhone")
    .map(f -> f.thenAccept(System.out::println))
    .toArray(size -> new CompletableFuture[size]);
CompletableFuture.allof(futures).join();
```

allof 工厂方法接收一个由CompletableFuture构成的数组，数组中所有CompletableFuture对象执行完成之后，返回一个CompletableFuture<Void> 对象。需要要等待最初Stream中的所有CompletableFuture对象执行完毕，对allOf 方法返回的CompletableFuture执行join操作是个不错的主意。

只执行任何一个执行完毕就不再等待 anyOf

