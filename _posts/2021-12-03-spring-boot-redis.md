---
layout: post
title: spring boot 整合redis
categories: reids
tags: [reids spring-boot]
excerpt: spring boot 整合redis
subtitle: spring boot整合redis
---

## 前言

redis是一个基于内存的key-value数据库，可以提供临时内容高效的存取功能，这次项目中决定用redis做token管理，处理登录过期，以及登录限制一类的功能，研究了两种池子lettuce和jedis的使用配置方式，在测试的时候发现了一些问题，记录一下。

## 版本
spring boot start 2.6.1

## lettuce pool 配置

### pom.xml
 ``` xml
       <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
 ```

### properties

```properties
    spring.redis.host=127.0.0.1
    # 默认redis数据库编号
    spring.redis.database=0
    spring.redis.password=123456
    spring.redis.port=6379
    spring.redis.timeout=3000
    spring.redis.lettuce.pool.max-active=8
    spring.redis.lettuce.pool.max-idle=4
    spring.redis.lettuce.pool.min-idle=2
    spring.redis.lettuce.pool.max-wait=-1
```


### RedisConfig.java

```java
@Configuration
public class LettuceRedisConfig {

    @Value("${spring.redis.host}")
    private String hostname;

    @Value("${spring.redis.password}")
    private String password;

    @Value("${spring.redis.database}")
    private int databaseIndex;

    @Value("${spring.redis.port}")
    private int port;

    @Value("${spring.redis.timeout}")
    private int timeout;

    @Value("${spring.redis.lettuce.pool.max-active}")
    private int maxActive;

    @Value("${spring.redis.lettuce.pool.max-idle}")
    private int maxIdle;

    @Value("${spring.redis.lettuce.pool.min-idle}")
    private int minIdle;

    @Value("${spring.redis.lettuce.pool.max-wait}")
    private int maxWaitTime;

    @Bean
    public RedisTemplate<String, Object> redisTemplate(LettuceConnectionFactory lettuceConnectionFactory) {

        //配置redisTemplate
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();

        // 配置连接工厂
        redisTemplate.setConnectionFactory(lettuceConnectionFactory);

        // 序列化方法没必要很深究,网上有比较多的说明 只要不乱码就可以
        redisTemplate.setKeySerializer(new StringRedisSerializer());//key序列化
        redisTemplate.setValueSerializer(new StringRedisSerializer());//value序列化
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());//Hash key序列化
        redisTemplate.setHashValueSerializer(new StringRedisSerializer());//Hash value序列化
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }

    @Bean
    public LettuceConnectionFactory lettuceConnectionFactory(GenericObjectPoolConfig genericObjectPoolConfig) {

        // 基础配置
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
        redisStandaloneConfiguration.setDatabase(databaseIndex);
        redisStandaloneConfiguration.setHostName(hostname);
        redisStandaloneConfiguration.setPort(port);
        redisStandaloneConfiguration.setPassword(RedisPassword.of(password));

        // connection timeout
        SocketOptions socketOptions = SocketOptions.builder().connectTimeout(Duration.ofMillis(timeout)).build();
        ClientOptions clientOptions = ClientOptions.builder().socketOptions(socketOptions).build();

        // 此处可以配置commandtimeout 和 shutdowntimeout
        // commandtimeout 大概是发出命令后等redis回复的最长时间 但是shutdowntimeout没有搞懂什么意思
        LettuceClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
                .poolConfig(genericObjectPoolConfig)
                //.commandTimeout()
                //.shutdownTimeout()
                .clientOptions(clientOptions)
                .build();

        LettuceConnectionFactory factory = new LettuceConnectionFactory(redisStandaloneConfiguration,clientConfig);
        return factory;
    }

    /**
     * GenericObjectPoolConfig 连接池配置
     *
     * @return
     */
    @Bean
    public GenericObjectPoolConfig genericObjectPoolConfig() {
        GenericObjectPoolConfig genericObjectPoolConfig = new GenericObjectPoolConfig();
        genericObjectPoolConfig.setMaxIdle(maxIdle);
        genericObjectPoolConfig.setMinIdle(minIdle);
        genericObjectPoolConfig.setMaxTotal(maxActive);
        genericObjectPoolConfig.setMaxWait(Duration.ofMillis(maxWaitTime));
        // 此处可以配置testonbrrow等线程池连接重试机制，如果不配置这些的话理论上可以不需要写这个redisconfig.java文件，springboot会自动生成所有需要的
        genericObjectPoolConfig.setTestOnBorrow(true);
        genericObjectPoolConfig.setTestOnCreate(true);
        genericObjectPoolConfig.setTestWhileIdle(true);
        genericObjectPoolConfig.setTestOnReturn(true);
        return genericObjectPoolConfig;
    }

}
```

### 测试

注入redisTemplate然后直接存值

![lettuce-result-ima](/assets/images/spring_boot_redis/lettuce-test-result.jpg)


### lettuce 调查中发现的问题

#### 问题1

在调查中，发现有不少人提到做压力测试的时候有堆外内存溢出的问题，不过我自己做压力测试的时候是没有发现有内存溢出导致程序挂掉的情况，可能是我单次存储的内容比较少的原因。

[内存溢出问题参考1](https://github.com/lettuce-io/lettuce-core/issues/705)
,[内存溢出问题参考2](https://blog.csdn.net/qq_42969135/article/details/116719825)

## jedis pool 配置

### properties

``` properties
spring.redis.host=127.0.0.1
spring.redis.database=0
spring.redis.password=123456
spring.redis.port=6379
spring.redis.timeout=3000
spring.redis.jedis.pool.max-active=8
spring.redis.jedis.pool.max-idle=4
spring.redis.jedis.pool.min-idle=2
```

### pom.xml

```xml
     <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
            <!-- 取出自带的lettuce依赖 -->
            <exclusions>
                <exclusion>
                    <groupId>io.lettuce</groupId>
                    <artifactId>lettuce-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!--Jedis依赖-->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
        </dependency>
```

### Jedis config java

```java
@Configuration
public class JedisRedisConfig {

    @Value("${spring.redis.host}")
    private String hostname;

    @Value("${spring.redis.password}")
    private String password;

    @Value("${spring.redis.database}")
    private int databaseIndex;

    @Value("${spring.redis.port}")
    private int port;

    @Value("${spring.redis.timeout}")
    private int timeout;

    @Value("${spring.redis.jedis.pool.max-active}")
    private int maxActive;

    @Value("${spring.redis.jedis.pool.max-idle}")
    private int maxIdle;

    @Value("${spring.redis.jedis.pool.min-idle}")
    private int minIdle;


    @Bean
    public JedisConnectionFactory connectionFactory() {

        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();

        // hostname
        config.setHostName(hostname);

        // port
        config.setPort(port);

        //database index
        config.setDatabase(databaseIndex);

        // password
        if (!StringUtils.isEmpty(password)) {
            config.setPassword(RedisPassword.of(password));
        }

        JedisClientConfiguration.JedisClientConfigurationBuilder jedisClientConfiguration = JedisClientConfiguration.builder();
        jedisClientConfiguration.usePooling().poolConfig(poolConfig());
        jedisClientConfiguration.connectTimeout(Duration.ofMillis(timeout));

        // init pool
        return new JedisConnectionFactory(config, jedisClientConfiguration.build());
    }

    public JedisPoolConfig poolConfig() {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMaxTotal(maxActive);
        jedisPoolConfig.setMinIdle(minIdle);
        jedisPoolConfig.setMaxIdle(maxIdle);
        jedisPoolConfig.setTestOnCreate(false);
        jedisPoolConfig.setTestOnBorrow(true);
        jedisPoolConfig.setTestWhileIdle(true);
        jedisPoolConfig.setTestOnReturn(true);
        return jedisPoolConfig;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        template.setDefaultSerializer(new StringRedisSerializer());
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        template.afterPropertiesSet();
        return template;
    }
}
```

### 测试

![jedis-test-tesult](/assets/images/spring_boot_redis/jedis-test-result.jpg)


### jedis 调查中发现的问题

#### 问题1

在测试中，本地环境没有问题，长时间的运行也可以，但是打成jar包上传到dev环境之后，做压力测试，一段时间过后会有很高的概率出现broken pipe问题 [参考](https://www.shangmayuan.com/a/3a71f03653754925a94efcb0.html) ,网上的朋友大多是在redisconfig中设置testonbrrow等字段解决问题，但是在添加之后这个问题并没有得到解决。




