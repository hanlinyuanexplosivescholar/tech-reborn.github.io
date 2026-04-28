业务场景：
       按用户+接口路径限流，限制每秒钟访问同一接口的次数。通过在redis缓存key计数，缓存失效时间为一秒

生产问题：
       个别缓存key，没有失效时间，导致用户访问一直处于限制状态


问题代码：
```
@Component
@Slf4j
public class RateLimitFilter implements GlobalFilter, Ordered {

    private static final String RATE_LIMIT_LUA_SCRIPT =
            "local current = redis.call('incr', KEYS[1]) " +
                    "if tonumber(current) == 1 then " +
                    "   redis.call('expire', KEYS[1], ARGV[1]) " +
                    "end " +
                    "return current";

    private static final int MAX_REQUESTS_PER_SECOND = 10;
    private static final String REDIS_KEY_PREFIX = "sr:settlement:userKeyResolver:rate_limit:";
    private static final List<String> EXCLUDED_PATHS = Arrays.asList("/sr-manager-service/v1/get/token/**", "/health");

    private final PathPatternParser parser = new PathPatternParser();

    @Autowired
    private ReactiveStringRedisTemplate redisTemplate;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("filter执行顺序 RateLimitFilter");

        String userToken = exchange.getRequest().getHeaders().getFirst("Authorization");
        log.info("接口限流 userToken：" + userToken);

        String path = exchange.getRequest().getURI().getPath();
        if (StrUtil.isNotBlank(path) && path.contains("/sr-manager-service/v1/get/token")) {
            return chain.filter(exchange);
        }

        String redisKey = REDIS_KEY_PREFIX + userToken + path;
        return rateLimit(redisKey, exchange, chain);
    }

    private Mono<Void> rateLimit(String redisKey, ServerWebExchange exchange, GatewayFilterChain chain) {
        return redisTemplate.opsForValue().setIfAbsent(redisKey, "1", Duration.ofSeconds(1))
                .flatMap(result -> {
                    if (result) {
                        return handleRequest(1L, exchange, chain);
                    } else {
                        return redisTemplate.opsForValue().increment(redisKey)
                                .flatMap(count ->
                                        redisTemplate.expire(redisKey, Duration.ofSeconds(1))  // 强制重新设置 TTL
                                                .then(handleRequest(count, exchange, chain))
                                );
                    }
                })
                .onErrorResume(e -> {
                    log.error("RateLimitFilter Redis operation failed for key: {}", redisKey, e);
                    exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
                    return exchange.getResponse().setComplete();
                });
    }


    private Mono<Void> handleRequest(Long count, ServerWebExchange exchange, GatewayFilterChain chain) {
        if (count > MAX_REQUESTS_PER_SECOND) {
            log.info("限流触发，用户请求过快，key: {}", exchange.getRequest().getURI().getPath());
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

```



修改后代码：

```
@Component
@Slf4j
public class RateLimitFilter implements GlobalFilter, Ordered {

    private static final String RATE_LIMIT_LUA_SCRIPT =
            "local current = redis.call('incr', KEYS[1]) " +
                    "if tonumber(current) == 1 then " +
                    "   redis.call('expire', KEYS[1], ARGV[1]) " +
                    "end " +
                    "return current";

    private static final int MAX_REQUESTS_PER_SECOND = 10;
    private static final String REDIS_KEY_PREFIX = "sr:settlement:userKeyResolver:rate_limit:";
    private static final List<String> EXCLUDED_PATHS = Arrays.asList("/sr-manager-service/v1/get/token/**", "/health");

    private final PathPatternParser parser = new PathPatternParser();

    @Autowired
    private ReactiveStringRedisTemplate redisTemplate;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("filter执行顺序 RateLimitFilter");

        String userToken = exchange.getRequest().getHeaders().getFirst("Authorization");
        log.info("接口限流 userToken：" + userToken);

        String path = exchange.getRequest().getURI().getPath();
        if (StrUtil.isNotBlank(path) && path.contains("/sr-manager-service/v1/get/token")) {
            return chain.filter(exchange);
        }

        String redisKey = REDIS_KEY_PREFIX + userToken + path;
        return rateLimit(redisKey, exchange, chain);
    }

    private Mono<Void> rateLimit(String redisKey, ServerWebExchange exchange, GatewayFilterChain chain) {
        return redisTemplate.execute(
                        RedisScript.of(RATE_LIMIT_LUA_SCRIPT, Long.class),
                        Collections.singletonList(redisKey),
                        Collections.singletonList("1")
                )
                .next()  // 获取 Flux<Long> 的第一个元素，转换为 Mono<Long>
                .defaultIfEmpty(0L)  // 防止 Redis 返回空值时报错
                .flatMap(count -> handleRequest(Long.valueOf(count.toString()), exchange, chain))
                .onErrorResume(e -> {
                    log.error("Redis 操作失败，限流过滤器异常，key: {}", redisKey, e);
                    exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
                    return exchange.getResponse().setComplete();
                });
    }

    private Mono<Void> handleRequest(Long count, ServerWebExchange exchange, GatewayFilterChain chain) {
        if (count > MAX_REQUESTS_PER_SECOND) {
            log.info("限流触发，用户请求过快，key: {}", exchange.getRequest().getURI().getPath());
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

```

总结问题原因：

修改前：
      使用Redis的基本命令组合实现 
执行流程： 先尝试设置键值对，带1秒过期时间 
如果设置成功，说明是第一次请求 
如果设置失败，则对计数器递增并重置过期时间 

       缺点： 存在并发问题
       需要多次Redis操作 
       性能相对较差 


修改后：
       使用Lua脚本实现 
执行流程： 通过单个原子操作执行Lua脚本 
脚本内完成计数递增和过期时间设置 

       优点： 原子性操作，避免并发问题
       只需一次Redis调用 
      性能更好 
      