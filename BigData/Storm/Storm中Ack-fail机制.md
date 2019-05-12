# Storm中Ack-fail机制

1.需要ack-fail时，为每一个Tuple生成一个messageID，这个messageID是用来标识你关心的Tuple，当这个Tuple被完全处理时，Storm框架会调用spout的ack方法，否则调用fail。至于消息是否进行重发，完全由程序编写决定。

```Java
MySpout{
    private Map buffer = new HashMap();
    spout.open();
    spout.nextTuple(){
        collector.emit();
        //发送的消息放置于缓存中
        buffer.put(msgId, messValue);
    }
    spout.outputFields();
    spout.ack(msgId){
        //消息移除
        buffer.remove(msgId);
    }
    spout.fail(msgId){
        String messValue = buffer.get(msgId);
        //消息重发
        collector.emit();
    }
    
}

MyBolt{
    bolt.execute(){
        // 手动调用ack方法
        collector.ack(tuple);
    }
}
```

2.在Spout有并发度的情况下，Storm会根据Tuple最开始的所属的Spout taskId，通知相应的SpoutTask。

3.在流式计算中的topology的bolt组件是可以配置多个的，在每个环节中，都需要bolt组件显示告诉Storm框架，自己对当前接受的Tuple处理完成。

<spoutTaskID，<RootId, ackValue >>

spout1 --->tuple(msgId, rootId) ---->bolt1 ---->collector.ack(tuple)

4.ack机制里面，发送两种类型的Tuple。一种是原始消息（DataTuple），另一种是ackTuple<RootId, tupleId>,DataTuple中会包含一个MessageID的对象。

**接收端层面**spout.emit(DataTuple(messageId(ackTuple))) ---->bolt1.execute(dataTuple)---->collector.ack(dataTuple) --->(从dataTuple中找到MessageID--->ackTuple---->Acker.execute(tuple))

**发送端层面**与Spout.emt()的同时，另一条线将ackTuple发送给Acker，ackTuple ----->Acker.execute(tuple)

