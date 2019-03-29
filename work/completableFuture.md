```java
CompletableFuture[] cfs = taskList.stream().map(integer -> CompletableFuture.supplyAsync(() -> cal(integer), executorService)

.thenApply(h -> Integer.toString(h))

.whenComplete((s, e) -> 

{System.out.println(s +  e);

 list.add(s)})).toArray(CompleteFuture[] :: new);

CompletableFuture.allOf(cfs).join();
```

