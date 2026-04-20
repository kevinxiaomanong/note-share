### 一、简介

在分布式系统中存在两个常见问题：

操作互斥性：如果不同的应用或是同一个应用的不同实例共享了一个或一组资源，那么访问这些资源时需要互斥来防止彼此干扰来保证一致性，这种情况下需要使用到**分布式锁**

操作幂等性：假设有如下场景，用户下单，服务端收到请求后处理，但在处理完成后服务器挂了，这时用户可能收到下单失败的提示，这时候用户重试，但实际之前的操作已经执行，导致重复下单，这就是在分布式系统中很常见的幂等性问题，所谓幂等性就是需要确保多次请求的结果是一致的，不会由于多次操作而产生副作用，这种场景下需要**幂等检查器**



### 二、使用

#### 2.1 SDK

```
@Bean
public DistributedLockService distributedLockService() {
    return new RedisDistributedLockService("your.namespace");
}

DLock dLock = distributedLockService.getLock("your.resource.key");
boolean locked = false;
try {
    locked = dLock.tryLock();
    if (locked) {
        // 成功获取资源
        // 处理业务逻辑
        // your code
    }
} catch (DistlockRejectedException e) {
    // 获取资源异常(分布式锁服务不可用)，根据业务场景进行降级处理
    // 比如说停止业务逻辑，或者正常执行业务逻辑
    // your code
} finally {
    if(locked) {
        try {
            dLock.unlock();
        } catch (DistLockIllegalMonitorStateException) {
            //根据业务实际场景处理
        }
    }
}
```



获取锁过程中发生的异常（例如服务不可用、网络异常）会抛异常，建议在业务逻辑的finally方法中，判断获取到锁后主动unlock以保证释放锁，这里为了使用户明确感知到当前线程解锁异常，会抛这个异常DistLockIllegalMonitorStateException：

1. tryLock失败时尝试解锁
2. tryLock成功，但是解锁时redis服务端认为锁已被别人持有，比如单次持锁超时；人工介入修改等情况



建议在finally判断locked后显示unlock



#### 2.2 本地化

不依赖测试环境，本地调试分布式锁



依赖credis 用户不需要申请NameSpace权限，只需要本地配置NameSpace的权限就可以直接使用



#### 2.3使用建议

过期时间：正常用户调用unlock后锁被释放，但某些特殊情况下，比如应用crash，网络异常，为了避免特殊情况下锁得不到释放加入了过期时间机制，默认30s，请确保指定的值大于预估的业务逻辑执行时间提高锁的可靠性



getLock：返回新的DLock对象，一旦完全unlock，锁被释放则该对象不可再次使用，如果想再次获取需要重新获取新的Dlock对象



**完全unlock与部分lock**

可重入锁的核心特性是：同一个线程多次获取锁不会阻塞，且需按获取次数释放，仅当释放次数=获取次数时，锁才真正释放，例如线程A重入获取2次锁

- 第一次释放：锁的重入计数从2-1 锁仍持有
- 第二次舒服，重入计数从1-0 锁真正释放





### 三、原理

如何基于redis  setnx实现可重入分布式锁，不能仅用线程ID（分布式场景下不同实例的线程ID可能重复），核心方案是：

锁的value设计：{唯一标识}:{重入计数}

其中 唯一标识：需保证跨实例、跨线程唯一，常用方案：

- 机器IP+进程ID+线程ID
- UUID+线程ID UUID保证实例唯一，线程ID保证实例内线程唯一

SETNX本身不支持重入，因为他无法体现“谁”占用了当前线程，需通过Lua脚本封装“判断-更新”的原子逻辑，避免并发问题

```
@Component
public class RedisReentrantLock {
    @Autowired
    private StringRedisTemplate redisTemplate;

    // 锁的默认过期时间（防止死锁）
    private static final long DEFAULT_EXPIRE_TIME = 30;
    // Lua脚本：获取锁（含重入逻辑）
    private static final String LOCK_SCRIPT = """
            local key = KEYS[1]
            local value = ARGV[1]
            local expire = ARGV[2]
            -- 1. 锁不存在：直接设置，重入计数=1
            if redis.call('exists', key) == 0 then
                redis.call('hset', key, value, 1)
                redis.call('expire', key, expire)
                return 1
            end
            -- 2. 锁已存在，且属于当前线程：重入计数+1
            if redis.call('hexists', key, value) == 1 then
                redis.call('hincrby', key, value, 1)
                redis.call('expire', key, expire)
                return 1
            end
            -- 3. 锁已存在，不属于当前线程：获取失败
            return 0
            """;
    // Lua脚本：释放锁（含重入计数递减）
    private static final String UNLOCK_SCRIPT = """
            local key = KEYS[1]
            local value = ARGV[1]
            -- 1. 锁不存在：直接返回（已释放）
            if redis.call('exists', key) == 0 then
                return 1
            end
            -- 2. 锁不属于当前线程：返回失败
            if redis.call('hexists', key, value) == 0 then
                return 0
            end
            -- 3. 属于当前线程：重入计数-1
            local count = redis.call('hincrby', key, value, -1)
            -- 4. 计数=0：删除锁；否则更新过期时间
            if count == 0 then
                redis.call('del', key)
                return 1
            else
                redis.call('expire', key, ARGV[2])
                return 1
            end
            """;

    // 生成线程的全局唯一标识（解决分布式线程ID重复问题）
    private String getThreadUniqueId() {
        // 方案1：IP+进程ID+线程ID（需获取本机IP和进程ID）
        String ip = NetUtils.getLocalIp(); // 自定义工具类获取本机IP
        long pid = ProcessHandle.current().pid(); // JDK9+获取进程ID
        long tid = Thread.currentThread().getId();
        return String.format("%s:%d:%d", ip, pid, tid);
        
        // 方案2：简化版（UUID+线程ID，无需获取IP/进程ID）
        // return UUID.randomUUID().toString() + ":" + Thread.currentThread().getId();
    }
}
```





## 幂等

**同一个操作，无论被执行 1 次还是多次，最终产生的业务结果完全一致，且不会对系统造成额外的、非预期的影响**。

在后端系统中，网络抖动、重试机制、用户重复操作、MQ 消息重复消费等场景几乎无法避免，幂等是保障系统稳定性的 “底线”：





| 手段       | 适用场景 | 核心原理 |
| ---------- | -------- | -------- |
| 唯一请求ID | 接口调用 |          |
|            |          |          |
|            |          |          |
|            |          |          |
|            |          |          |



采用唯一请求ID（幂等号）+Redis+防重表 组合方案

1. 调用方生产全局唯一幂等号（UUID/雪花ID），随请求传递
2. Redis做前置校验，保障性能，防重表（Mysql唯一索引）做最终兜底
3. 结合乐观锁保证订单状态一致性



```
redis aop
/**
 * 幂等注解：标记需要幂等的方法
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
    // 幂等号的请求参数名（比如请求中传递的idempotentId）
    String key() default "idempotentId";
}

@Component
@Aspect
@Slf4j
public class IdempotentInterceptor {
    @Autowired
    private StringRedisTemplate redisTemplate;

    // Redis中幂等号的过期时间（30分钟，覆盖大部分业务超时场景）
    private static final long EXPIRE_TIME = 30 * 60;

    @Pointcut("@annotation(com.example.demo.annotation.Idempotent)")
    public void idempotentPointcut() {}

    @Around("idempotentPointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        // 1. 获取请求中的幂等号
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Idempotent annotation = signature.getMethod().getAnnotation(Idempotent.class);
        String idempotentKey = getRequestParam(joinPoint, annotation.key());
        if (StringUtils.isBlank(idempotentKey)) {
            throw new IllegalArgumentException("幂等号不能为空");
        }

        // 2. Redis前置校验：SETNX（不存在则设置，原子操作）
        Boolean success = redisTemplate.opsForValue().setIfAbsent(idempotentKey, "EXECUTED", EXPIRE_TIME, TimeUnit.SECONDS);
        if (Boolean.FALSE.equals(success)) {
            // 幂等号已存在，说明重复请求
            log.warn("重复请求，幂等号：{}", idempotentKey);
            return Result.fail("操作已执行，请勿重复提交");
        }

        // 3. 执行原方法
        try {
            return joinPoint.proceed();
        } catch (Throwable e) {
            // 执行失败，删除幂等号（允许重试）
            redisTemplate.delete(idempotentKey);
            throw e;
        }
    }

    // 从请求参数中提取幂等号（适配POST/GET请求）
    private String getRequestParam(ProceedingJoinPoint joinPoint, String key) {
        // 简化实现：实际可适配@RequestParam/@RequestBody等
        Object[] args = joinPoint.getArgs();
        for (Object arg : args) {
            if (arg instanceof OrderCreateRequest) {
                return ((OrderCreateRequest) arg).getIdempotentId();
            }
        }
        return null;
    }
}
```



订单业务

```
@Service
@Slf4j
public class OrderService {
    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private OrderUniqueMapper orderUniqueMapper; // 防重表Mapper

    /**
     * 创建订单（幂等实现）
     * @param request 下单请求（含幂等号、用户ID、商品ID等）
     */
    @Idempotent(key = "idempotentId") // 标记需要幂等
    @Transactional(rollbackFor = Exception.class)
    public Result<Long> createOrder(OrderCreateRequest request) {
        Long userId = request.getUserId();
        Long productId = request.getProductId();
        String idempotentId = request.getIdempotentId();

        // 1. 防重表兜底（MySQL唯一索引：uk_idempotent_id）
        OrderUniqueDO uniqueDO = new OrderUniqueDO();
        uniqueDO.setIdempotentId(idempotentId);
        uniqueDO.setBizType("ORDER_CREATE");
        uniqueDO.setCreateTime(new Date());
        try {
            orderUniqueMapper.insert(uniqueDO);
        } catch (DuplicateKeyException e) {
            // 唯一索引冲突，说明重复创建
            log.warn("防重表拦截重复下单，幂等号：{}", idempotentId);
            return Result.fail("订单已创建，请勿重复操作");
        }

        // 2. 乐观锁扣减库存（库存表：product_stock，字段：id, product_id, stock, version）
        int updateCount = orderMapper.decreaseStock(productId, 1);
        if (updateCount == 0) {
            throw new BusinessException("库存不足");
        }

        // 3. 创建订单（状态机控制：初始状态为待支付）
        OrderDO orderDO = new OrderDO();
        orderDO.setOrderNo(generateOrderNo()); // 生成订单号
        orderDO.setUserId(userId);
        orderDO.setProductId(productId);
        orderDO.setStatus(OrderStatus.PENDING_PAYMENT.getCode()); // 待支付
        orderDO.setCreateTime(new Date());
        orderMapper.insert(orderDO);

        return Result.success(orderDO.getId());
    }

    // 生成唯一订单号（雪花ID）
    private String generateOrderNo() {
        return SnowflakeIdGenerator.generateId();
    }
}
```

库表：

```
-- 防重表（核心：唯一索引uk_idempotent_id）
CREATE TABLE `order_unique` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键',
  `idempotent_id` varchar(64) NOT NULL COMMENT '幂等号',
  `biz_type` varchar(32) NOT NULL COMMENT '业务类型（如ORDER_CREATE）',
  `create_time` datetime NOT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_idempotent_id` (`idempotent_id`) COMMENT '幂等号唯一索引'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 库存表（乐观锁：version字段）
CREATE TABLE `product_stock` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键',
  `product_id` bigint NOT NULL COMMENT '商品ID',
  `stock` int NOT NULL COMMENT '库存数量',
  `version` int NOT NULL DEFAULT '0' COMMENT '版本号（乐观锁）',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_product_id` (`product_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 乐观锁扣减库存SQL
UPDATE product_stock 
SET stock = stock - 1, version = version + 1 
WHERE product_id = #{productId} AND stock > 0 AND version = #{version};
```



调用方：

```
// 调用方生成幂等号（UUID），随请求传递
String idempotentId = UUID.randomUUID().toString();
OrderCreateRequest request = new OrderCreateRequest();
request.setIdempotentId(idempotentId);
request.setUserId(10086L);
request.setProductId(20001L);
Result<Long> result = orderService.createOrder(request);
```



注意点：

1. 幂等号要设置过期时间，避免一直占用内存，防重表可以定时任务清理历史数据
2. 业务执行失败时，需删除Redis中的幂等号，允许重试，但防重表不删除，仍能拦截重复请求
3. 幂等号需全局唯一，统计幂等拦截次数，若某时段拦截量突增，可能重复异常

幂等需多层兜底（Redis+防重表），且异常时需合理处理幂等号























