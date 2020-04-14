# 一、id缓存器

### 1、整体流程

* 核心**setIdCacheMap**方法注入所有的**IdCache**实现类，统一放入一个容器中

  ```java
  @Autowired(required = false)
  public void setIdCacheMap(List<IdCache> idCaches) {}
  ```

* **replaceAttributeValue**将缓存中对应的属性值进行注入：

  * **replaceEmptyFields**将Model中为null的属性字段替换为相应的Id

  * 通过Id列表为key先从Redis里面取缓存，**redisTemplate.opsForValue().multiGet(idSet)**

  * 如果缓存不存在，**cacheAttributeValue**方法调用相应类型的**IdCache**的**getAttributesByIds**

    查询到相应的值，然后设置到缓存中**redisTemplate.opsForValue().multiSet(allAttributes)**

  * 进行正则匹配，将为空的属性替换成缓存中对应的值

* 采用@ThService+Aop统一增强接口，处理返回的VO（**@AfterReturning**）,将VO中为空的属性填充相应缓存中的值

  ```java
  JSON.parseObject(IdCacheHelper.replaceAttributeValue(JsonUtil.toJSONStringWithNull(result)), PageResponse.class)
  ```

### 2、使用方法及使用示例

* 在Service实现类上添加@ThService

```java
@ThService
public class TestThServiceImpl {}
```

* 在接口返回的VO中，添加缓存Model中的属性字段以获取缓存

```java
缓存Model：
public class UserIdCacheInfo {

    @ApiModelProperty(value = "用户id")
    private String userId;

    @ApiModelProperty(value = "用户名")
    private String username;

    @ApiModelProperty(value = "手机号")
    private String phone;
}

public class UserInfoVO {
    
    @ApiModelProperty(value = "用户id")
    private String userId;

    @ApiModelProperty(value = "用户名")
    private String username;

    @ApiModelProperty(value = "手机号")
    private String phone;
}

// 注意！！：一定要设置相应的Id，上述就是userId
new UserInfoVO().setUserId("....");

// 注意！！：如果是myUserName和myPhone如何获取到缓存呢？
// 在相应的IdCache实现类中的getFieldPrefix()方法新增需要的前缀"my"
@Override
public List<String> getFieldPrefix() {
    return Lists.newArrayList("my");
}
```

* 如果想在代码中手动获取缓存，则只需直接调用**IdCacheHelper.replaceAttributeValue()**方法

```java
JSON.parseObject(IdCacheHelper.replaceAttributeValue(JsonUtil.toJSONStringWithNull(result)), PageResponse.class)
```

### 3、功能如何扩展

1、添加缓存Model

```java
public class UserIdCacheInfo {

    @ApiModelProperty(value = "用户id")
    private String userId;

    @ApiModelProperty(value = "用户名")
    private String username;

    @ApiModelProperty(value = "手机号")
    private String phone;
}
```

2、添加IdCache实现类

```java
@Component
public class UserIdCache implements IdCache {

    @Autowired
    private UserDOMapper userDOMapper;

    @Override
    public String getIdCacheType() {
        return IdPrefix.USER_ID.getIdPrefix();
    }

    @Override
    public List<UserIdCacheInfo> getAttributesByIds(List<String> ids) {
        return userDOMapper.listCacheInfoByIds(ids);
    }

    @Override
    public String getIdFieldName() {
        return getIdCacheModelClass().getDeclaredFields()[0].getName();
    }

    @Override
    public Class getIdCacheModelClass() {
        return UserIdCacheInfo.class;
    }

    @Override
    public List<String> getFieldPrefix() {
        return Lists.newArrayList("recommend");
    }

    @Override
    public Map<String, IdCacheCleaner> getIdCacheCleaner() {
        Map<String, IdCacheCleaner> map = Maps.newHashMap();
        map.put("POST-/jobSeeker/resume_build", (Object[] args) -> Lists.newArrayList(UserContext.getUserId()));

        return map;
    }
}
```

# 二、身份验证

1、校验流程

注册生成校验JWT必要的tokenSalt盐；登录校验账户密码，生成token并把tokenSalt盐放入缓存；使用拦截器拦截请求，解析请求头中的token，处理得到user基本信息，并使用签名对token进行验证是否有效

**注册**：在数据库生成用户基本信息的同时存入tokenSalt盐

```java
String secret = UUID.randomUUID().toString();
userDO.setTokenSalt(secret);
```

**登录**：

​	1、通过用户名查询User信息

​	2、比对密码

​	3、取出tokenSalt盐，把盐和用户基本信息存入Redis缓存，并设定过期时间

​	4、处理是否允许重复登录

​		①、允许重复登录：不更改盐，重新生成一个新的有效token返回给前端

​		②、不允许重复登录：更改盐让原有的token失效，并把新的盐更新到数据库和缓存中，然后用新的盐去生成				新的token返回给前端

```java
UserDO userDO = userService.getUserByTarget(passwordLoginReq.getUser());
if (!StringUtils.equals(userDO.getPassword(),
                        PasswordUtils.encodePassword(passwordLoginReq.getPassword(), 							userDO.getUserId()))) {
    // 密码错误
    throw new BizException(UserErrorInfo.PASSWORD_ERROR);
}

String key = REDIS_USER_PREFIX + userId;
RedisUtils.hPutAll(key, userDTO);
RedisUtils.expire(key, tokenExpire, TimeUnit.MILLISECONDS);

String token;
if (!switchConfiguration.isEnableRepeatLogin()) {
    // 不允许重复登录, 登录之后其他token作废
    token = updateTokenAndSalt(userId);
} else {
    token = JwtUtils.generateJwt(userId, getUserJwtExpire(userId), tokenSalt, null);
}
```

**拦截器**：

​	1、处理URL，对不要验证的URL进行放行

​	2、解析请求头token，把JWT中的有效负载payload提取出来

​	3、从payload的sub信息中获取userId

​	4、通过userId去Redis缓存中获取用户数据

​		①、如果存在，重置缓存过期时间

​		②、如果不存在，从数据库中查询，更新缓存用户数据

​	5、使用签名对token进行校验是否有效

​	6、如果验证的token在续期时间内，就对其进行续期

```java
if (requestUrl.equals("/") || requestUrl.contains("swagger") || requestUrl.contains("/error") || requestUrl
            .contains("/csrf") || request.getMethod().equals("OPTIONS")) {
    log.warn("不做权限校验的url: {} => {}", request.getMethod(), requestUrl);
    return true;
}

// 3. 资源类型不验证
if (handler instanceof ResourceHttpRequestHandler) {
    // todo: 一般是请求的接口不存在时到这里, 到这里时不是标准格式返回, 需不需改变
    return true;
}

// 4. 不用检查权限的api 直接通过
boolean isFreeLogin = ((HandlerMethod)handler).hasMethodAnnotation(FreeLogin.class);
if (isFreeLogin && !((HandlerMethod)handler).getMethodAnnotation(FreeLogin.class).value()) {
    // 不用获取用户信息的免登录接口
    return true;
}
```

