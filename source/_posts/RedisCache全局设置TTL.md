---
title: Redis Cache 设置TTL
tags:
  - java
  - redis
  - Spring Cache
categories:
  - 技术集
thumbnail: "/images/night.jpg"
excerpt: false

---
------
#### Redis Cache 设置TTL

全局设置redis cache ttl 的方法

### 配置
```java
@Slf4j
@Configuration
@EnableCaching
@EnableConfigurationProperties(RedisProperty.class)
public class CacheConfig {

    /**
     * 创建并配置一个 {@link RedisConnectionFactory} 实例。
     *
     * @param redisProperties 包含 Redis 连接属性的对象，如主机名、端口、数据库编号和密码。
     * @return 配置好的 {@link RedisConnectionFactory} 实例，用于与 Redis 服务器建立连接。
     */
    @Bean
    public RedisConnectionFactory redisConnectionFactory(RedisProperty redisProperties) {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration(redisProperties.getHost(), redisProperties.getPort());
        redisStandaloneConfiguration.setDatabase(redisProperties.getDatabase());
        redisStandaloneConfiguration.setPassword(redisProperties.getPassword());
        return new LettuceConnectionFactory(redisStandaloneConfiguration);
    }

    /**
     * 创建并配置 RedisCacheManager 实例。
     *
     * @param redisConnectionFactory Redis 连接工厂，用于创建与 Redis 服务器的连接
     * @return 配置好的 RedisCacheManager 实例
     */
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        return new RedisCacheManager(
                RedisCacheWriter.nonLockingRedisCacheWriter(redisConnectionFactory),
                // 默认策略，未配置的 key 会使用这个
                this.getRedisCacheConfigurationWithTtl(0),
                // 指定 key 策略
                this.getRedisCacheConfigurationMap()
        );
    }

    /**
     * 获取Redis缓存配置映射。
     * <p>
     * 该方法会扫描指定包下的所有类，并通过反射工具获取这些类中的缓存配置信息，最终生成一个包含缓存名称和对应配置的映射表。
     *
     * @return 包含缓存名称和对应配置的映射表
     */
    private Map<String, RedisCacheConfiguration> getRedisCacheConfigurationMap() {
        Map<String, RedisCacheConfiguration> redisCacheConfigurationMap = new HashMap<>();

        // 扫描指定包下的所有类
//        Set<Class<?>> clsList = new HashSet<>();
//        try {
//            CacheConfigScan cacheConfigScan = AnnotationUtils.findAnnotation(B2BManageAuthApplication.class, CacheConfigScan.class);
//            if (cacheConfigScan != null) {
//                // 获取扫描包路径
//                String[] value = cacheConfigScan.value();
//                // 获取class
//                Arrays.stream(value).distinct().forEach(t -> {
//                    clsList.addAll(ClassUtil.scanPackage(t));
//                });
//            }
//        } catch (Exception e) {
//            log.error("scan cache config error", e);
//        }
//        // 通过反射工具获取到需要扫描的包下的所有的class
//        validAnnotation(clsList, redisCacheConfigurationMap);
return redisCacheConfigurationMap;
}

    /**
     * 获取带有指定过期时间的 Redis 缓存配置。
     *
     * @param seconds 缓存项的过期时间（秒）
     * @return 配置好的 RedisCacheConfiguration 对象
     */
    private RedisCacheConfiguration getRedisCacheConfigurationWithTtl(Integer seconds) {

        return RedisCacheConfiguration.defaultCacheConfig().serializeValuesWith(
                        RedisSerializationContext.SerializationPair
                                .fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .computePrefixWith(t -> removeTtlPre(t) + ":")
                .entryTtl(Duration.ofSeconds(seconds));
    }

    /**
     * 移除缓存名称中的 TTL 前缀。
     *
     * @param cacheName 缓存名称字符串，可能包含 TTL 前缀
     * @return 处理后的缓存名称字符串，去除了 TTL 前缀
     */
    private String removeTtlPre(String cacheName) {
        String[] split = cacheName.split("#");
        if (split.length >= 2) {
            return Arrays.stream(split).skip(2).collect(Collectors.joining());
        }
        return cacheName;
    }

    /**
     * 验证并处理带有 @Cacheable 注解的方法，提取缓存配置并存储到 redisCacheConfigurationMap 中。
     *
     * @param clsList                    包含需要验证的类的集合
     * @param redisCacheConfigurationMap 存储缓存配置的映射，键为缓存键，值为缓存配置
     */
    private void validAnnotation(Set<Class<?>> clsList, Map<String, RedisCacheConfiguration> redisCacheConfigurationMap) {
        if (CollUtil.isEmpty(clsList)) {
            return;
        }
        clsList.forEach(cls -> {
            Method[] methods = cls.getDeclaredMethods();
            for (Method method : methods) {
                // 方法上有@Cacheable
                Cacheable cacheable = method.getAnnotation(Cacheable.class);
                if (Objects.isNull(cacheable)) {
                    continue;
                }
                // 按照定义好的规则截取出过期时间, 并进行设置
                String cacheKey = cacheable.value()[0];
                String[] split = cacheKey.split("#");
                int cacheTtl;
                if (split.length >= 2) {
                    cacheTtl = Integer.parseInt(split[1]);
                } else {
                    cacheTtl = 0;
                }

                redisCacheConfigurationMap.put(cacheKey, this.getRedisCacheConfigurationWithTtl(cacheTtl));

            }
        });
    }
}
```


