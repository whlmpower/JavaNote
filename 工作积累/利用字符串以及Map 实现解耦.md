利用字符串以及Map 实现解耦

1.注册

```java
protected final Map<String, AbstractCallBackHandler> callbackMap = Maps.newHashMap();

    public void register(AbstractCallBackHandler handler) {
        AbstractCallBackHandler existedHandler = callbackMap.get(handler.acceptedType());
        if (existedHandler != null && !existedHandler.getClass().equals(handler.getClass())) {
            throw new IllegalArgumentException("Handler重复注册: " + handler.acceptedType());
        }
        callbackMap.put(handler.acceptedType(), handler);
    }
```

2. 获取

```java
String type = urlParam.getType();
        JobStage jobStage = JobStage.getByLabel(failedStage);

        AbstractCallBackHandler handler = callbackMap.get(type);

```



