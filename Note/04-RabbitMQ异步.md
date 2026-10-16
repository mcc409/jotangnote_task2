# RabbitMQ 异步处理

## 一、功能说明

把笔记的增删改操作改成异步：
- 接口收到请求后**立即返回**，不等数据库操作完成
- 操作封装成消息丢进 RabbitMQ
- 后台消费者从队列取消息，执行真正的数据库操作

## 二、核心代码

### RabbitConfig.java（定义队列）

```java
@Configuration
public class RabbitConfig {
    public static final String NOTE_QUEUE = "note.queue";

    @Bean
    public Queue noteQueue() {
        return new Queue(NOTE_QUEUE, true);
    }

    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

### NoteConsumer.java（消费者）

```java
@Component
public class NoteConsumer {

    @Autowired
    private NoteMapper noteMapper;

    @RabbitListener(queues = RabbitConfig.NOTE_QUEUE)
    public void handle(NoteMessage message) {
        System.out.println("【MQ 消费者】收到消息：" + message.getOperation());

        switch (message.getOperation()) {
            case "CREATE":
                noteMapper.insert(message.getNote());
                break;
            case "UPDATE":
                noteMapper.update(message.getNote());
                break;
            case "DELETE":
                noteMapper.deleteById(message.getNote().getId());
                break;
        }
    }
}
```

### NoteController 改成发消息

```java
@PostMapping
public Note create(@RequestBody Note note,
                   @RequestHeader(value = "userId", required = false) Integer userId) {
    note.setUserId(userId);
    note.setCreatedAt(LocalDateTime.now());
    // 不再直接写数据库，而是发消息
    rabbitTemplate.convertAndSend(RabbitConfig.NOTE_QUEUE,
        new NoteMessage("CREATE", note));
    return note;
}
```

## 三、测试流程

调用新增/修改/删除接口后，看 IDEA 控制台：

```
【MQ 消费者】收到消息：CREATE
【MQ 消费者】收到消息：UPDATE
【MQ 消费者】收到消息：DELETE
```

./imgs/mq-consumer.png

## 四、注意事项

接口返回值里的 id 是 null —— 这是异步的正常现象。

因为接口在消息投递后立即返回，此时数据库还没执行插入，所以 Java 对象里的 id 还没被回填。等消费者处理完后，数据库里才有这条数据